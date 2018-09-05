---
layout:     post
title:      Unexpected things when using Pact to do consumer driven testing（C#版本）
subtitle:   C#版本绕坑记
date:       2018-07-24
author:     leihui
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Pact
    - 契约测试
---
# 前言

>最近在用Pact做契约测试，这里有个Step by step的workshop很不错:[Example .NET Core Project for Pact Workshop](https://github.com/tdshipley/pact-workshop-dotnet-core-v1),但也碰到一些问题，记录下来，帮助别人快速搭建Pact环境。

首先，契约测试和传统的API测试有相似点，那两者区别是什么？
契约测试主要解决了几个API测试不好解决的问题。例如：
设想两个场景
1.后端没有开发好，前端想开始开发，怎么办？
传统的API测试，碰到这种情况，直觉是想在前端用一个mock server来做，但接着又回碰到一系列问题，mock server中的request和response应该是consumer定义，还是provider定义？定义好的mock server,如果provider端没有按照这个定义去实现怎么快速发现等等。
而Pact作为测试框架，比较优雅的解决了这一系列的问题。

2.传统的API测试失败了，到底是后端修改代码造成的，还是使用者代码造成的？（特别是后端环境比较复杂时候，定位时间会比较长）
在传统API测试中，需要调查原因分析，花费时间。现在用了契约测试，如果是后端代码造成的，那么消费者端的CI上是测试通过的，因为消费者端用的是mock server。而后端CI会挂，因为后端replay契约文件时候会报错。这样马上就定位出造成错误的地方了。

例子：假设consumer定义好了request和response，在consumer端写单元测试时候，会根据自己定义的response的内容做进一步处理，假设这个进一步的处理是一个1000行的复杂业务逻辑代码。好了，现在单元测试是通过了，过了一个月后，发现这个单元测试失败了，那么开发可以先查看request和response发生改变没有，如果没有发生改变，就可以断定是自己的业务逻辑代码处理被修改了，导致单元测试挂了。

如果是在以前用postman等做api测试时候，类似上面的场景，一个月后，开发也发现这个单元测试挂了，那么会有两种可能，一种是consumer端的那个1000行的业务逻辑代码被改变了，也有可能是provider端修改了response的内容导致测试失败，这时候就要花费更多的时间去调查原因。


1.运行consumer端的测试，有时候不会更新生成的json格式的契约文件。重新运行一遍就会做更新。

2.在consumer端的xunit中定义了mockserver,如下代码：
```
    _mockProviderService.Given("some state")
                                .UponReceiving("To search houses")
                                .With(new ProviderServiceRequest
                                {
                                    Method = HttpVerb.Post,
                                    Path = /api/somehouse,
                                    xxxxxxx....,
                                    Body = new 
                                    {
                                        origin = "Xian",
                                        xxxx....
                                    }
                                })
                                .WillRespondWith(new ProviderServiceResponse
                                {
                                    Status = 200,
                                });
```
这些代码已经定义了request和response。如果认为定义好后，运行xunit测试，就能生成json格式契约文件，是不行的。
必须接着真实给mockserver按照上面定义格式的path等发送一个请求后，才能生成契约文件。例如执行下面的代码
```
    var resultBodyText = sendPostToMockServer(new StringContent("{xxxxx}}", Encoding.UTF8, "application/json"));
```
其中sendPostToMockServer方法的实现如下：
```
     var result = ConsumerApiClient.sendPost("/api/somehouse", _mockProviderServiceBaseUri, content).GetAwaiter().GetResult();
```
目前感觉这一步有些多此一举，为什么需要给mockserver发送一次请求，而且请求中的path需要和给_mockProviderService定义的request中的path对应上，然后Pact才会成json文件？
_mockProviderService不过是按照之前定义的request和response去做事情，不用真实的发送请求，它也完全有做够的信息去生成json文件的。

3.我们在给api发送请求，request的body中常常需要设定信息。在C#的Pact版本中，不能把这些json信息直接copy到代码中使用，例如body中信息类似：

```
{  
   "origin":"Xian",
   "passengers":[  
      {  
         "type":"ADT",
         "discountCode":""
      }
   ]
}
```
在C#code中需要改成：
```
{
    origin = "MEL",
    passengers = new List<Object>(){
        new {
            type = "ADT",
            discountCode = ""
        }
    },
}
```
如果body中的信息比较多，这个手工修改量让人汗颜。建议Pact提供一个直接API,用来可以支持去使用从body里copy过来的json.

4.在provider端，做契约文件的回放时候，我需要在每次发送请求中，先去访问并得到一个token,然后把token加入到request的header中，这时候，执行回放前，Pact会提示说：“你已经修改了header，因此契约测试不能完全保证运行的结果是对的。”这点很赞。

5.在Consumer端定义state时候，这个state会在provider端作为一个key，去映射到不同的方法去做测试数据的预处理。就是线面的Given方法中定义的内容。
```
_mockProviderService.Given("some state")
```
接着在provider端的ProviderState方法中定义一个api去处理这个state。
```
pactVerifier.ProviderState($"{_pactServiceUri}/provider-states")
```
问题是目前只看到ProviderState这一个方法去处理state，传入的参数必须是个url，导致这个workshop需要在provider端专门启动一个server去处理。实际上Pact可以做成不用url的方式，比如在provider测试类中定义startup方法的方式去做这件事情，显得更轻量级一些。

6.Pact的文档相对成熟开源框架的文档少很多，网上的例子也少。有些文档中的具体细节步骤描述的不够清楚。

7.我在本机搭建了docker环境，这里面启动了pact broker,同样发现了偶发的执行单元测试后，不上传契约文件到broker的现象，再执行一次就可以上传了。

8.现在本机试验docker中的pact broker时候，是按照这个页面中去做https://github.com/DiUS/pact_broker-docker，里面有一个section是“Running with Docker Compose”。打开Docker Compose的yml文件后，有两行代码如下：
```
    - ./ssl/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    - ./ssl:/etc/nginx/ssl
```
中间的冒号是标示把本机的文件映射到docker中启动的nigix服务器的对应目录下。问题来了，这个ssl目录中的文件在哪里，实际上在https://github.com/DiUS/pact_broker-docker的ssl目录中。

9.启动docker后，它执行了compose的yaml文件，会分别启动pact broker，数据库服务器，nigix，执行完毕到最后会打印两个error，暂时没时间看是什么原因，但是到目前还没有影响broker的使用。
```
postgres_1    | 2018-07-23 09:29:31.864 UTC [1135] ERROR:  relation "schema_migrations" does not exist at character 27
postgres_1    | 2018-07-23 09:29:31.864 UTC [1135] STATEMENT:  SELECT NULL AS "nil" FROM "schema_migrations" LIMIT 1
postgres_1    | 2018-07-23 09:29:31.996 UTC [1135] ERROR:  relation "schema_info" does not exist at character 27
postgres_1    | 2018-07-23 09:29:31.996 UTC [1135] STATEMENT:  SELECT NULL AS "nil" FROM "schema_info" LIMIT 1
```
契约文件上传到pact broker如下图：
![Pact Broker](/img/pact_broker_1.jpg)

10.Docker中pact broker的默认账户密码是"username"和"password"，在consumer端用下面的代码push到broker中
```
private void pushToPactBroker()
        {
            var pactPublisher = new PactPublisher("http://xxxxxx", new PactUriOptions("username", "password"));
            pactPublisher.PublishToBroker(
                @"..\..\..\..\..\pacts\consumer-provider.json",
                "2.1.3");
        }
```
在provider端用下面的代码去读取json文件：
```
IPactVerifier pactVerifier = new PactVerifier(PactVerifierConfig);
            pactVerifier.ServiceProvider("Provider", ProviderUri)
                .HonoursPactWith("Consumer")
                .PactUri(@"http://xxxxx/pacts/provider/Provider/consumer/Consumer/latest")
                .Verify();
```
