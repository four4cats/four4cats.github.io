---
title: iOS 使用 Jenkins 实现持续集成
date: 2020-03-29 14:43:39
description: Jenkins - iOS 出坑指南
author: TimorYang
categories: 
 - iOS
tags: iOS, Jenkins, fastlane, infer, CI/CD
---

# iOS 使用 Jenkins 实现持续集成

### Jenkins 初探
>每当功能开发周期结束后，就会陷入 **Fix bug** 和**打包**的无限循环中，我当时就在想，如果把打包这样的重复性工作交给电脑来做,我就有可以利用节省下来的这些碎片时间来研究其他有趣的事情。早年在中国移动工作的时候就接触到了 Jenkins，但一直没有去深入了解过，这一次算是在坑里趟了一遍，就写下了这篇 Jenkins 之 iOS 出坑指南。

接下来阐述下我的思路，其实很简单，我就是使用 Jenkins 把目前工作中所用到的流程串起来了。
1. 获取 Github(也可以是基于 Git 的其他服务商，或者是 SVN) 的最新代码
2. 编译代码
3. UT & UITest
4. Static code analysis
5. Archive(这一步不仅仅是打包，使用 [fastlane](fastlane) 做到了，自动上传 AppStore，TestFight 分发)

虽然看上去只有简简单单的 5 步，但确实可以用“一步一个坑”来形容，一点也不顺利。
现在思路，流程都想好了，那么接下来就是考虑怎么用 Jenkins 实现了。
### 实现方案
仔细品品上面的所说的这五个步骤，其实它们之间有着依赖关系，如果说你在拉去最新代码的时候就失败了，下面 2，3，4...的这些步骤就没有必要去执行了，所以我们可以把这 5 个步骤对应成 Jenkins 里面的 5 个 Job，第一个 Job 的功能就是负责拉取最新代码，第二个 job 就是编译代码，以此类推。同时我又希望如果某一个 Job 执行失败了，会通过邮件或者其他的方式通知到我们，让我们及时排除问题，让整个流程又可以继续跑下去。当跑完整个流程后，我还希望持续集成产出的IPA(或者是 jar，apk 等等)可以放到我们指定的 FTP 服务器下面，并通过邮件告知我们打包完成可以开始新的测试了。
放到指定的地方，是因为当某个版本出来问题，可以及时找到对应的dSYM（[Debug symbol](https://en.wikipedia.org/wiki/Debug_symbol)）文件，符号位 crash 文件，提高了定位问题的效率。当然归纳的好处并不止这一点，我这里就不展开叙述了。
#### Job 依赖
为了实现我们刚刚所说的依赖关系，我查看了 Jenkins 文档，Jenkins 里面有个**上下游**的概念，举个例子 Job A 执行成功需要触发 Job B，那么 Job A 就是 Job B 的上游，反过来，Job B 就是 Job A 的下游。

*++怎么实现上下游呢？++*
##### Plan A
在 Jenkins 里面每一个 Job 的配置里都有**Build after other projects are built**这个选项，我们在这里填写依赖的上游 Job 名称。
使用方法如下
![plana.png](http://130.211.252.198:8090/upload/2020/03/plan-a-7b60753eba744f98bc5c8d976e20ad55.png)

##### Plan B
Jenkins Plugin: [Parameterized Trigger](https://plugins.jenkins.io/parameterized-trigger/)
使用这个插件，我们在建立依赖关系的同时还能往下游传递参数让下游 Job 可以根据不同的条件执行不同的任务。
使用方法如下：
Job A
![planb1.png](http://130.211.252.198:8090/upload/2020/03/plan-b-1-f976bd38e3b14493a16006e1c7406871.png)![planb2.png](http://130.211.252.198:8090/upload/2020/03/plan-b-2-ed97ecda7dc54f5bae187167e7f7819b.png)
Job B
![planb3.png](http://130.211.252.198:8090/upload/2020/03/plan-b-3-b0d4224f082340bdb52e65ac12e17a2c.png)

以上两种方案，按需使用。
这里根据需求，我选用的是 Plan B，因为 Jenkins 的每一个 Job 的 workspace 都是独立的，而我设计的整个持续集成的流程其实作用的对象是同一个，so，我希望所有的 Job 都是在同一个 workspace 里面运行，理所当然我需要把 workspace 的绝对路径作为参数传给下游 Job，来达到让所有的 Job 作用的对象都是同一个。
现在，我们可以很容易的将上面所说的这 5 个步骤串起来，接下来我们需要实现每一个 Job 需要做的事情。
#### Step 1 获取最新代码
通用配置
* 检查 Jenkins 的设置里面是否正确配置了 Git 环境，如果没有正确配置，会报**待填**这个错误
* 配置 Git 超时时间，如果项目工程比较大，加上网络不是很顺畅，很容易出现默认的 10 分钟超时拉取失败的错误
* 配置 webhook (Push, Pull Request, more...)

特殊需求
* Jenkins 需要拉取多个分支的代码

使用方法如下：
1. 配置项目地址
2. 配置仓库地址，如果是私有仓库，需配置账号密码或者 SSH 密钥
![Jenkinsgit1.png](http://130.211.252.198:8090/upload/2020/03/Jenkins-git-1-c902b04cf05447fda63a83dce6a90976.png)
3. 这里我使用 Job 依赖的 Plan B 方案，传递**branch_ref**和**workspace**两个参数
4. 触发下游 Job

#### Step 2 编译代码
这一步，我们使用 [xcodebuild](https://developer.apple.com/library/archive/technotes/tn2339/_index.html) 命令来编译我们的项目工程代码。这里使用的比较简单，更多关于 xcodebuild 的使用方法，请看官方文档。
``` shell
# 如果是含有 Pods 工程
pod install
xcodebuild -workspace xxx.xcworkspace -scheme xxx
# 如果不含 Pods 工程
xcodebuild -project xxx.xcodeproj -scheme xxx
```
这一步主要是为了确认当前最新的代码是否能编译通过，如果不能编译通过，Jenkins 会通过 Email 的方式通知开发者，让开发者来解决当前最新代码无法编译通过的问题。
这个 Job 不会产生任何的参数，所以我们只需要将上游 Job 传递过来的参数，继续传给下游项目。

#### Step 3 UT & UITest
*<u>目前我们项目的单元测试和 UITest还在 coding 阶段，所以这一步暂不实现。</u>*

#### Step 4 Static code analysis
> Static code analysis 目前主流的工具有两个，一个是 [OCLint](http://oclint.org) 还有一个是 [Infer](https://fbinfer.com)(Facebook 开源)。

我这里选用了 Infer 来做 Static code analysis，并配合 Jenkins PMD Plugin 来做分析报告的可视化。
``` shell
# Infer 安装
brew install infer
# Start static code analysis
# Pods工程
infer run -- xcodebuild -workspace xxx.xcworkspace -scheme xxx -configuration Debug -sdk iphonesimulator
# 非 Pods 工程
infer run -- xcodebuild -project xxx.xcodeproj -scheme xxx -configuration Debug -sdk iphonesimulator
```
配置 PMD Plugin
![JenkinsPMD.png](http://130.211.252.198:8090/upload/2020/03/Jenkins-PMD-85b8ef1ecc1c4fbb82bbc4b2e5a8e904.png)

当分析结果出来后，使用 Jenkins 将当前的结果通过 Email 发送给开发同学，通知开发同学尽快修复存在隐患的代码。

同样的，这里我们还是继续将上游传递过来的参数，继续传给下游项目，不做任何修改。
#### Step 5 Archive
> fastlane, App automation done right. The easiest way to build and release mobile apps.
fastlane handles tedious tasks so you don’t have to.

我们利用强大的开源社区提供的工具**fastlane**来完成最后一步打包和分发。
安装 fastlane
``` shell
# Using RubyGems
sudo gem install fastlane -NV

# Alternatively using Homebrew
brew install fastlane
```
初始化项目
``` shell
# Objective-C
fastlane init

# swift
fastlane init swift
```
配置 fastlane file
**这里建议第一次接触 fastlane 的小伙伴，仔细阅读 fastlane 的[文档](https://docs.fastlane.tools)。**
最终的执行命令
```shell
# 上游传递的参数 branch_name
if [ branch_name = "master"]
then;
	# 提交苹果商店
	fastlane release
fi

if [ branch_name = "release"]
then;
	# 提交到 TestFlight
	fastlane beate
fi

# 如果是 dev 分支，就什么都不做
```

**特别注意**
当你在 Jenkins 里执行 fastlane时，可能会遇到
`While executing gem ... (Errno::EACCES)
    Permission denied @ rb_sysopen - /Users/edz1/.rvm/rubies/ruby-2.6.3/lib/ruby/gems/2.6.0/wrappers/bin-proxy`
这个错误，那么你需要配置下 Ruby 环境。
![Jenkinsrubyversion.png](http://130.211.252.198:8090/upload/2020/03/Jenkins-ruby-version-0345f92c7e8044098151c658d274482f.png)
And，这个问题就解决了，你可以继续下面的步骤了。

当 Archive Job 成功后，Jenkins 会通过 Email 通知技术部门的全体小伙伴打包成功，可以开始新的一轮测试了。

至此，我们整个自动化流程就配置结束了，我们可以利用这个时间去研究其他的事情了。🍻🎉 enjoy it。

这里记录下我在 fastlane 配置 match 证书遇到的坑

**Git Storage on GitHub**

If your machine is currently using SSH to authenticate with GitHub, you'll want to use a git URL, otherwise, you may see an authentication error when you attempt to use match. Alternatively, you can set a basic authorization for match:

`match(git_basic_authorization: '<YOUR KEY>')`
文档里提示使用上面的语法来进行 Github Personal access tokens 的配置
但是按照文档的配置，死活都不能成功，我在 fastlane 的 issues 中找到了对应的解决方案
* https://github.com/fastlane/fastlane/issues/15560
> I realized that this was not my problem. I didn't find in Fastlane documentation that I need to encode the token like Base64.encode64("myusername:mypersonaltoken") and use it as my basic auth token. Would be good if we update the docs and add this tip.

👆上面应该是 fastlane 的开发者回复的，让我们使用`git_basic_authorization 'Base64.encode64("myusername:mypersonaltoken")'`来配置 match file，尝试了一波后，还是失败了，然后我接着查 issues, 找到了这个[解决方案](https://github.com/fastlane/fastlane/issues/15560#issuecomment-548437763)，将**Base64.encode64** 改成**Base64.strict_encode64**，And it works 🎉。
### 写在结尾
如果你也在尝试使用 CI/CD 来做持续交互，遇到了一些问题，希望你不要放弃，当你从坑里爬出来后，你就会获得相关的经验，帮助你成长，当你一个人被一个问题难住很久的时候，不妨先方向手头的事情，走一走，捋一捋思路，说不定就茅塞顿开了。你也可以在下面给我留言，我们一起解决🍻。