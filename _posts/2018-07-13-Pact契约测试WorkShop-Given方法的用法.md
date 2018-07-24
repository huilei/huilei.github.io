---
layout:     post
title:      Pact契约测试workshop-Given方法的逻辑（C#版本）
subtitle:   C#版本
date:       2018-07-13
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Pact
    - 契约测试
---
# 前言

>在[Example .NET Core Project for Pact Workshop](https://github.com/tdshipley/pact-workshop-dotnet-core-v1)中详细介绍了**C#版Pact的例子**,例子很棒！
这篇文章只说一下例子中MockServer的Given方法，看看Given中传递的参数是怎么一步步被使用的。实际上在这个Workshop中，由于使用了Given中的参数，增加了很多行代码去处理Given中给的参数。

1.首先，打开[工程目录]/pact-workshop-dotnet-core-v1/YourSolution/Consumer/tests/ConsumerPactTests.cs文件，在单元测试ItHandlesInvalidDateParam中可以看到这行代码：

``
_mockProviderService.Given("There is data")

``

这里就是我说的Given中给的参数"There is data"。我开始以为这个参数只是一个类似description的描述文本，实际上不是的。他会决定在契约测试的Provider端走哪一个代码分支去做测试的预制条件。（预制条件在这个例子中，是增加一个文件，或者删除一个文件）。

2. 在Consumer端按照Workshop中的步骤，执行一遍单元测试后，会在[工程目录]/pact-workshop-dotnet-core-v1/pacts下生成Json文件，Json文件中的内容是完全根据单元测试中定义生成的，可以看到Given("There is data")这句对应着Json文件里的"providerState": "There is data"这行。

3.




