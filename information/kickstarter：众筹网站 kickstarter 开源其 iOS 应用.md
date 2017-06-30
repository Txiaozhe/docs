### kickstarter：众筹网站 kickstarter 开源其 iOS 应用

[原文](https://gold.xitu.io/entry/5851de7c8d6d8100658e8133/view)

[GitHub](https://github.com/kickstarter/ios-oss)

#### 获取和运行

* 下载并安装Xcode
* 将该仓库克隆至本地
* 运行`$ make bootstrap`安装工具和依赖
* 运行`$make test-all `构建、测试跨平台应用

#### 探索一些有趣的事

* `Screenshots` 目录中拥有将近500张不同语言、不同设备和边缘的情况下我们确保真实状态的屏幕截图。
* 我们使用 Swift Playgrounds 来进行开发和风格的迭代，app中大多数重要的页面都有一个符合标准的playground使我们可以实时看到各种各样的设备、语言和数据，浏览我们对这些playground的收集。
* 我们使用视图模型作为一种轻量级的方法来隔离副作用，并维护一个核心功能。我们把这些作为输入信号到输出信号的纯映射，并对他们做大量的测试，包括本地化、可访问性和事件跟踪的测试。


#### 项目依赖

> 第一部分

* [Prelude](https://github.com/kickstarter/Kickstarter-Prelude): Foundation of types and functions we feel are missing from the Swift standard library.
* [KsApi](https://github.com/kickstarter/ios-ksapi): Models and reactive networking layer for fetching data from Kickstarter’s API.
* [ReactiveExtensions](https://github.com/kickstarter/Kickstarter-ReactiveExtensions): A collection of operators we like to add to ReactiveCocoa.

> 第三部分

* [AlamofireImage](https://github.com/Alamofire/AlamofireImage)
* [Argo](https://github.com/thoughtbot/Argo)
* [FBSnapshotTestCase](https://github.com/facebook/ios-snapshot-test-case)
* [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)

#### 捐献

我们打算把这个项目作为一个教育资源：我们很高兴能分享我们的胜利，错误和ios开发的方法，我们在开放的工作。我们的主要重点是继续改善我们的用户符合我们的路线图的应用程序。

提交反馈和报告错误最好的办法就是打开一个GitHub的issues，提交时请务必包括您的操作系统、设备、版本号和步骤，重现报告的错误。请记住。所有参与者都将遵循我们的代码规范。

#### 代码规范

我们的目标是分享我们的知识和发现，因为我们在一个安全和开放的空间每天工作为社区改善我们的产品。我们像对待生活一样地对待工作，鼓励人们学习和成长，并给予和接受积极的、建设性的反馈。并且我们也保留删除或禁止任何违反此基础的行为的权利。

#### 许可

```shell
Copyright 2016 Kickstarter, PBC.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```











