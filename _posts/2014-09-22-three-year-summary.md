---
layout: post
title: "成长，没那么迫切"
keywords: ["algorithm", "POJ"]
description: "三年工作小结"
category: "thinking"
tags: ["summary","work"]
comments : true
---


早就想敲一篇工作总结了，却因一直以为自己只是一个初出茅庐的小屁孩而迟迟不动手。再换工作时，猛然发现自己已然入行三年，想来也可笑又难以启齿，竟是混混沌沌地打了三年酱油。

**2011年6月29日 大三 上海某软件公司研究院**

 被这家公司录用挺意外的，说实话大学前两年的确有做梦想进这家公司，但时至大三在对自己的能力进行了各方面评估之后，自我感觉能去个珠海某公司，那已然是上天最大的眷顾了，而收到这家公司的实习录用通知时，更是激动了好几个晚上没睡好，鄙视我吧～～～～
 好吧，当时是这样的情况，整个大学基本上玩算法但玩得很烂，而除了算法，当时是啥都不会，真的啥都不会。老大倒是挺坦然地接受我这个状态了，既然啥都不会，就从头开始学呗。

**LNMP** 

先是装了半个月的机器，说出去真丢人，大学愣是没接触过Linux，刚开始连apt-get是啥都不知道。记得当时搭得第一套服务是Linux+mysql+php+nginx,作为一个连windows都不怎么熟悉的超级屌丝，每次编译安装都几乎要了小命，阴影至今还在……至于shell脚本是啥，不知道……

**Solr**


老大对我还是挺有耐心的，当我终于学会了什么叫编译安装和apt-get之后，开始让我学着搭建solr服务，这个阶段还算顺利，几乎把官网上所有文档看了个遍，只可以当时还不知道怎么学习开源软件，文档是看了，愣是没花时间去看源码，暴殄天物……

**python**

会搭搜索服务后，从哪儿获取搜索资源呢？那就猥琐一点去网上抓吧。看来看去用python最简单，于是开始用python编写简单的抓取网页数据脚本，倒也挺开心顺手。
然后，用python搞点其它事情吧，好像做了个转码服务啥的，用gearman来做任务分发，尼玛记得回学校前夜为了装这玩意我还搞到凌晨2:00，真TM敬业啊。

**Hadoop**

有了数据后，自然是学做数据分析，能看得出来当时老大是一心想把我往大数据方向培养，可惜当时我的天赋情商实在太低。HDFS最终还是搭建起来了，mapreduce 也跑起来了，但mahout最终还是没学扎实，现在只记得有itemBase 和  userBase两种推荐算法，大致实现是构建模型，然后几个矩阵放在一起做计算，借助于矩阵运算天然的分布式优势，套用mapreduce自然飞快……
 
**Objetive-C**

好吧，这完全是玩去了，现在只记得Mac台式机的屏幕非常酷，老被我用来照镜子，还有就是objective－C的代理机制实在太讨厌，把一个活生生的算法爱好者弄得晕头转向……

**Cloud Storage**

11年正是国内云计算活跃起来的时候，作为一个资深糊涂屌丝程序员，自然跟风而动。调入云计算存储项目组后，先是学习元数据存储，知道了什么叫 r+w>n,了解动静分离，接触了maven。

接着是学着用awk grep分析日志，用birt做报表。
再接着学习用php做前段，学会了简单的javascript，css,html.
然后是维护各种语言的SDK，包括php,ruby,c/c++,听着也许很牛逼，实际情况是玩这些的人都跑了，作为爱公司第一人，当然舍身取义顶上，伟大吧巴拉巴拉

**GVStream**

玩点云计算相关的东西呗。游戏那边说好像做美女游戏主播挺来钱的，一帮人坑哧坑哧开搞。

先是学了点视频的基础知识，就码率音频之类的，然后就是学用ffmpeg转码，为了支持直播，把视频切成一个个小的ts文件，丢到云存储，再用云分发分发。

学会了用java spring框架写restful API，用highcharts做报表，知道了mongodb 、HttpLiveStream,明白了啥是播放列表，虽然最后项目不幸夭折了，倒也乐在其中，学了不少东西。

**Operation**

老大说，不会运维的开发不是好开发，老大也说，云计算三分开发七分靠运维。作为一个立志要成为云计算达人的人，求老大教我运维，虽然最后只玩了半个月，但在这个过程中，知道了什么叫自动化部署脚本，知道LVS，heart Beat,opentsdb.

**2013年5月17日 离职**

公司忽然抽风说不让玩云计算了，通通被拉去做视频。作为云计算的忠实粉丝，当然不干，就这样，离开了第一家单位。

**2013年5月31日 北京某电商公司**

还是熟悉的云计算，初到公司，先是敲了一个月的报表，说好听点，叫运营监控系统哈哈。然后是学习leveldb源码，接着遍开始玩go,陆陆续续先后开发了分布式消息系统，分布式块存储，分布式对象存储元数据子系统，期间开始玩mysql，学习mysql内核。

**2014年7月17日 离职**

北京一年，似乎每天都很忙，却总感觉少了些什么，或许是离家太远？打好包袱，踏上归途，以为自己会不后悔，猛然发现，自己竟是那么怀念那段北漂的日子……

**2014年8月6日 杭州 新的开始**

或许，成长没有那么迫切。不过还是加油，好姑娘每天都会很努力嘛！


 
