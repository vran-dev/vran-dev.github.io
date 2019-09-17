---
title: "从30分钟到1分钟 - SBT的update耗时优化记录"
date: 2019-09-17
tags: [Scala, SBT]
---


# 从30分钟到1分钟 - SBT的update耗时优化记录

# 前言

公司有项目是基于 Scala 编写的，与之配套的构建工具是 SBT , 它是 **Simple Build Tool** 的缩写，虽然我觉得它一点也不简单。

这个项目有一个很大的痛点就是刷新依赖 （对应 SBT 的 update）非常之耗时，可以参见下图：

![image-20190916164031838](img/image-20190916164031838.png)

可以看到该操作耗费了1266秒，近半个小时，更新期间啥也做不了 （风扇还呼呼呼的转，温度蹭蹭蹭的长），该项目又依赖了很多内部服务，这些服务可以说是几乎每天都在升级，可以想象每天耗费在这方面的时间是何其多。

人生苦短，在刷新了几次之后，我再也受不了这漫长的等待时间，于是开始了这漫漫的优化之路。

> 工欲善其事必先利其器



## Round 1: 十八般武艺齐上阵

不知道大家碰见这种问题会怎么做，我反正是二话不说打开 Google 直接搜： **SBT 依赖下载慢**。

还别说，有共鸣的人还不少， 总结了下几乎都是以下的解决方案

> 1. 添加代理
> 2. 添加国内镜像源

我肯定不是源的问题啊，我司用的私有仓库，既然私有jar都下载下来了，肯定是走的私有仓库啊。

翻了几页，没有满意的答案，也试了几个方案，也没啥用。

看来还是得自己从问题的根源开始找起啊......

为了保险起见， 我还是先排查一下是不是镜像问题， 在项目的 `build.sbt` 配置文件中添加有有公司内部的仓库地址

```java

lazy val commonSettings = Seq(
	//....

  // ... 私有仓库
  resolvers := {Resolver.url("xr-ivy-releasez", new URL("http://nexus.xxxx.com/repository/ivy-releases/"))(Resolver.ivyStylePatterns) +: resolvers.value},
  resolvers := { {"xr-maven-public" at "http://nexus.xxxx.com/repository/public/"} +: resolvers.value},
  // ....
)

```

此时我忽然想到一种情况：难道是默认走的公共库，在公共库找不到依赖才会走私有库 ？

为了验证猜想，我使用 wireshark 抓包进行分析，过滤器指定协议 http (因为仓库是走的http)

> 还可以指定 ip.src 和 ip.dst 从而使得数据包更加符合我们的要求

![image-20190916201433355](img/image-20190916201433355.png)



然后打开 sbt shell 进行 update 操作

![image-20190916201505347](img/image-20190916201505347.png)



观察抓包结果

![image-20190916204017434](img/image-20190916204017434.png)

发现访问的都是 `/repository/public/***` 的请求，对应的 Host 也是我司的私有库，这说明配置是生效了的，而且都是从私有仓库进行下载。

但是我也发现了一些404的请求

![image-20190916204123537](img/image-20190916204123537.png)

好吧，是我司的私有库没有 `ivy-release` 相关的东西，果断将对应的配置去掉，省去没必要的请求。

虽然走了私有库，但是我每次刷新都会请求仓库，这就不符合道理了，难道 SBT 连基本的依赖缓存都没有 ？



## Round 2: 从半小时到五分钟

对抓到的数据包进行再次过滤，只看 `Request`，发现所有的请求的依赖版本都是 `SNAPSHOT`, 参见下图

![image-20190916204940489](img/image-20190916204940489.png)

这说明 SBT 是有缓存的，因为不是 SNAPSHOT 的都没有走网络请求，可是为什么 `SNAPSHOT` 每次都走网络请求呢？ 难道是 `SNAPSHOT` 不会被缓存？

既然 SBT 是基于 Ivy 的，那就从 Ivy下手。

我在Ivy 的官网（http://ant.apache.org/ivy/history/2.0.0/settings/caches.html）找到了下面的一个关于**缓存**的表格：

| Attribute           | Description                                                  | Required                                                     |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| default             | the name of the default cache to use on all resolvers not defining the cache instance to use | No, defaults to a default cache manager instance named 'default-cache' |
| **defaultCacheDir** | a path to a directory to use as default basedir for both resolution and repository cache(s) | No, defaults to **.ivy2/cache** in the user's home directory |
| resolutionCacheDir  | the path of the directory to use for all resolution cache data | No, defaults to defaultCacheDir                              |
| repositoryCacheDir  | the path of the default directory to use for repository cache data. **This should not point to a directory used as a repository!** | No, defaults to defaultCacheDir                              |

注意关键字 `defaultCacheDir`， 这个就是 Ivy 的缓存目录，对应路径为用户目录下的 `.ivy2/cache`。

我的是 mac， 对应目录就是 `~/.ivy/cache` , 果不其然，进入该目录查看一下：

![image-20190916203654004](img/image-20190916203654004.png)

在 `~/.ivy/cache` 下发现了很多依赖库的目录， 下面就需要验证一下有没有缓存 `SNAPSHOT` 的版本了， 以我司的 `user-client 4.1.2-SNAPSHOT` 为目标进行查找：

![image-20190916204726415](img/image-20190916204726415.png)

目录中明明有缓存 `SNAPSHOT` 的啊，为什么不走本地缓存呢 ？

这没办法了，只能去 SBT 官网找答案了，在官网文档找到了 [Dependency Management](https://www.scala-sbt.org/1.x/docs/Dependency-Management-Index.html) ，看名字似乎和依赖管理有关， 

其中的 [Cached-Resolution](https://www.scala-sbt.org/1.x/docs/Cached-Resolution.html) 似乎和缓存相关，在这里面我又发现了关键词 **SNAPSHOT and dynamic dependencies**，这其中的部分内容如下：

> When a minigraph contains either a SNAPSHOT or dynamic dependency, the graph is considered dynamic, and it will be invalidated after a single task execution. Therefore, if you have any SNAPSHOT in your graph, your experience may degrade. 

说的是依赖关系图如果有 `SNAPSHOT` 版本，会导致某个子依赖关系缓存失效，这会导致用户体验不好。

原来是 SBT 的 `SNAPSHOT` 不走缓存，SBT将其定义为动态依赖。

还好 SBT 算有良心，它 提供了一个配置去解决这个问题，如下

> The tradeoff is probably a longer resolution time if you have many remote repositories on the build or you live away from the severs. So here’s how to disable it:
>
> ​	**updateOptions := updateOptions.value.withLatestSnapshots(false)**

大概意思就是，加上这个配置使得 `SNAPSHOT` 的解析结果得以缓存，似乎问题很容易解决嘛。

不多说，加上配置，然后打开了 SBT 和 WireShark, 执行了 update, 看着这熟悉的画面出现在我眼中

![image-20190916205702243](img/image-20190916205702243.png)

此刻我是奔溃的！贼子安敢欺我！

正在我想静静之际，SBT 刷新完成，我一不小心瞄了一眼，耗时居然只有以前的1/4 了？

![image-20190916212240522](img/image-20190916212240522.png)

我靠，怎么肥四（回事）？不是没生效吗，怎么时间缩短了这么多？

为了确保不是眼花，我又重启刷新了几次，发现耗时相差无几。

不死心的我又去看了下文档，原来是我对这个配置理解错了，这个配置的缓存是和SBT进程相关的！

sbt启动后第一次执行update后，会缓存所有的依赖解析信息。而我的项目是有4个子项目的，每个子项目都共同依赖了 `service `模块， 该模块维护着几乎所有的依赖。

当第一个项目 update 后，其他三个项目 update 时都会直接走缓存了，这也是为什么耗时只有最开始1/4。

真是柳暗花明又一村啊......

## Round 3：从五分钟到一分钟

虽然现在时间只要以前的1/4了，可还是要5分钟啊，这绝对不是一个可以将就的数字！

而且还有另外一个非常重要的原因，因为穷！此话怎讲？因为 SBT 一直启动着太耗内存了，我这可怜的 8G 可得省着点儿。可是停掉 SBT，缓存就得重新构建了......

穷, 激发了我的进一步探索: 为什么每次启动都要去拉取快照呢 ？ 能不能只在依赖的版本有更新的时候再去拉取呢 ？

因为快照在不改变版本的情况下是可以重复发布的，区分同版本不同快照就只能按照时间戳来了。

SBT 无法确定本地的快照是最新的，所以每次启动都会去仓库拉取最新快照。

这样看来，问题和快照版本有关系，解决方案有两个

1. 不使用 `SNAPSHOT`
2. 把 `SANPSHOT` 版本当做正式版用

方案1就是直接把所有的快照改为正式版就可以了，但是几十个服务，找各个团队沟通我也嫌麻烦，就直接pass了。

那就只有用方案2了，这个方案就是不拉取最新版的快照，只拉取升级了版本的快照。可这个得官方支持才行，没办了，又只能扎进文档里面去。

还好黄天不负有心人啊， 在官方文档 [Cache And Configuration](https://www.scala-sbt.org/1.x/docs/Dependency-Management-Flow.html#Caching+and+Configuration) 一节找到了相关内容

> When `offline := true`, remote SNAPSHOTs will not be updated by a resolution, even an explicitly requested update. This should effectively support working without a connection to remote repositories. Reproducible examples demonstrating otherwise are appreciated. Obviously, update must have successfully run before going offline.

简单的说就是加上配置 `offline := true` 配置以后只有版本发生变化，并且该版本没有被缓存才会去仓库拉取依赖，也就是和正常版一样的效果了。

吹了这么多，最终还是得遛一遛才知道是不是有货。

加上配置再次进行 update 并抓包， 没有任何的请求到达仓库了

![image-20190916213212002](img/image-20190916213212002.png)再来看看最终的更新时间

![image-20190916213133659](img/image-20190916213133659.png)

只需要一分钟不到，嗯，再也不怕内存不够了......

其实吧，这个时间也不是最优的，可是到这一步了，再进行优化的话提升已经没有之前那么大了，但是要付出的成本却不比之前小。



## 写在最后

最后从30分钟到1分钟实际上就是在 `build.sbt` 加了两行配置

```scala
offline := true,
updateOptions := updateOptions.value.withCachedResolution(true).withLatestSnapshots(false)
```

问题的根源其实也很好定位，无非就是网络或者缓存的问题，最难的反而是找到解决方案，这其中我就走了很多弯路，我甚至去看了 SBT 中 **Resolver** 的源码，现在看来绝对是跑偏了。

其实整个解决问题的方法并没有多么高深莫测，甚至可以说是无聊至极，因为大部分时间都是在文档中去寻找并验证其配置。

不过还是那句话：**工欲善其事必先利其器**

## 参考

1. [sbt Reference Manua](https://www.scala-sbt.org/1.x/docs/)
2. [sbt 源码](https://www.scala-sbt.org/0.13/sxr/)
3. [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
4. [Apache Ivy Documentation (2.0.0)](http://ant.apache.org/ivy/history/2.0.0/index.html)