---
layout:     post
title:      Unexpected things when using Pact to do consumer driven testing（C#版本）
subtitle:   C#版本绕坑记
date:       2018-07-13
author:     leihui
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Pact
    - 契约测试
---
# 前言

>最近在用Pact做契约测试，这里有个Step by step的workshop很不错。[Example .NET Core Project for Pact Workshop](https://github.com/tdshipley/pact-workshop-dotnet-core-v1),但也碰到一些问题，记录下来，帮助别人快速搭建Pact环境。

1.运行consumer端的测试，有时候不会更新生成的json格式的契约文件。重新运行一遍就会做更新。
2.在consumer端的xunit中定义了mockserver,如下代码：
{{{
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
}}}
这些代码已经定义了request和response。如果认为定义好后，运行xunit测试，就能生成json格式契约文件，是不行的。
必须接着真实给mockserver按照上面定义格式的path等发送一个请求后，才能生成契约文件。例如执行下面的代码
{{{
    var resultBodyText = sendPostToMockServer(new StringContent("{xxxxx}}", Encoding.UTF8, "application/json"));
}}}
其中sendPostToMockServer方法的实现如下：
{{{
     var result = ConsumerApiClient.sendPost("/api/somehouse", _mockProviderServiceBaseUri, content).GetAwaiter().GetResult();
}}}
目前感觉这一步有些多此一举，为什么需要给mockserver发送一次请求，而且请求中的path需要和给_mockProviderService定义的request中的path对应上，然后Pact才会成json文件？
_mockProviderService不过是按照之前定义的request和response去做事情，不用真实的发送请求，它也完全有做够的信息去生成json文件的。






