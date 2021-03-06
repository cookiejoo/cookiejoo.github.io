---
title: 你在教我做事？
layout: post
author: 曹德高
tags: [self-cultivation]
categories: self-cultivation
---

![1111](/images/2021-12-16-self-cultivation/1.webp)

。。。我本来写了很多感慨，现在全删了，无非是一些苦口婆心的劝诫，想了想这些只是我自己的自以为而已。

我想写这篇文章很久了，我写的时候就在想，我有什么资格教人这些东西，你也只是个普通小职员，又不是领导、总监、经理。你有这些资质吗？搞的好像有多能似的。

最后只能妥协的我以一个30多岁的老油条、混职场多年的人，诉说一些个人成长心得吧，因我刚入社会职场那会也是靠自身多年的经历感悟如何让自己快点成长、与德配位。如果那会的我能有个老同志教我一些经验，可能我现在又不一样了。亦或许我当时根本看不起这些教条，依然我行我素的走自己认为的路。

*温馨提示：本文拟自身IT工作生涯来讲述。*

### 职业动机

先讲个案例，说一老总招了两个高学历的毕业生作为培养对象，A是清华本硕连读毕业的，B是某211本科后硕士考到清华毕业的。两人都很好，脑子都很灵活，做事都有条理。某天公司出差老总就带他们两个人也一起，A就跟老总住一间，他们两个人也有机会在一起深度聊聊天，老总发现A对很多事情都很在行，比如某电脑配件啊、某音乐设备啊等等，聊的头头是道，但是他仅仅是对他感兴趣的事在这方面上投入精力。也就是说其实他对工作并不放在最重要的位置，A对职业上的规划其实没投入太多时间的。而B却跟A相反，以至于后来老总离开了这家公司后，B后来去了一家央企一进就当干部培养，A还在原来那家公司当职员。

这故事不是说A、B谁不好，并非这类的评论，两个人是在对待职业规划上有个分歧，A觉得生活才是最重要的，工作只是生活上的一小部分而已，B觉得是职业很重要，我要在这个年龄段不停的往上走，我的目标要更高的职位或利益等等。我听到这个例子确实很有感触的，现今的社会有点家底的家庭其实是有很多，他们的后代其实没必要努力跟谁谁谁争个高下，他们的前一辈积累的财富其实够他们用了，自己在职业这条每人都要走的道路上发挥自己该有的价值就很好了。我认为这个是个人幸福感值的满意度问题。有人觉得普普通通过一辈子就够了，有人觉得我一定要住多大的房子，开多豪的车才算功成名就。

这就是在职业上的动机，那你又属于什么样动机呢？

### 如何提高工作效率

可不是每个人的父辈都有家底的，拼不过自己的爹，那你只能乖乖的拼自己。

我相信每个人都希望能一天只工作8小时，到点就下班，但在我们IT届的词典里根本不可能。那你要看清一点：你是真的忙还是假的忙。你是对工作没有分好类分好优先级，还是其实你在做给你的老板看的，如果是后者的话那就在自残了，有这些时间还不如写写技术博客，看看开源项目的代码，做做面试题等等打发时间。如果你是在业务部门，你离职后其实什么都带不走，除非你跳槽的另一家跟你做一模一样的项目，要不然换个业务其实都是零开始。而你花了时间积累的话，你可能还有第二春的备选方案，所以说后者就是自残。

经过我这些年的摸爬滚打领悟，提高工作效率的办法有三：

1：列清单

2：分优先级

3：有总结

我遇到很多同事，作为一个IT工作者5年以上甚至8到10年往上了，做事真的没有一点头脑经验可言，要说你刚毕业三年我可以理解，毕竟年轻有资格犯错。但历经多年的工作者一点章法都没有，其实都是不愿花时间思考的结果。

我强烈推荐《清单革命》这本书给大家，里面讲述的案例应用是非常广泛的，非常有效的，我称之为可复制的成功案例。我的领导在跟我一对一复盘的时候跟我说，为什么别人能对一件有挑战的事情能做成，其实是有方法论的，他做成这件事要检查什么依赖什么，按这个列好的清单去一项一项过，把该考虑的点都例检一次方向上就不会错。而《清单革命》这本书以飞机航班出行例检预案这个例子讲述的方法我一直都在使用着，一直都坚信的成功案例。

我所在的组是架构组，我们组会评审公司每一个架构变更（虽然不是我去评审，只是有些相关的我才参与），但每一个架构评审邮件我都会自己看一遍，第一看别人的架构怎么做、实现什么、怎么考虑，第二看自己是否可以看出问题，第三看别人的评审意见，看别人是怎么发现架构漏洞的。最后我发现了端倪，我们的架构千奇百怪，一些你见过的没见过的，你怎么去有效的评审？就是靠列好的清单，如你的数据到了什么量级要用什么资源、你规划的这个数据库是否能支撑未来增长需求、你的数据是否包含了敏感信息需要安全介入、你的调用链是否走的繁琐重复等等。你需要一个架构基本法来支撑你的检查项，我们也有架构基本法，所以清单是多么重要，你没见过的东西你都能驾驭。

说起架构基本法我就想到一件去年的事，我去参加一个外部技术研讨会，有家公司的技术开发者提出一个问题问在座的各位在一张表很大的情况下数据索引也做了，但是有时候没有太大的效果该怎么办？有人说考虑换种数据库，有人说这个索引不合理，有人说要分库分表。他说数据量也来越大了，超过了多少亿的数据非常难动。我的意见是，这个问题的解决方案其实不在技术上，而且在决策上，我们每家公司都在制定自己的架构研发体系，架构法案其实就是在解决这类数据庞大的问题，最早的阿里技术手册就是很有用的参考法案，你们只要制定了法案，那么到了某一个量级的数据你就要开始做分库分表迁移数据了，而不是要考虑什么索引最优单机什么数据库能抗的住诸如此类极端问题。

做事要分等级，这个事项我在刘润的五分钟商学院课程里面学到了很多，还有刘军的课程也讲了很多案例，你的todo列表中，这件事不干活今天不干是不是会死人？你要拎清楚了。我每次收到我组长分配的任务我会问一声这件事的紧急程度是什么？交付日期在什么时候？如果你不问清楚，等交付的时候你又吐不清楚，那你可能这一整年都白干了，要知道人是有刻板映像的，你做好99件事，但只要有1件事做不好，你的得分都不会是99，而是69。

做完事要有总结，至少你做完一个里程碑要总结一次，不关是你突破了什么技术也好解决了某项业务难题也好怎么做事提高了效率也会，要有输出，否则你忙了一年到头好像事做了很多，又好像啥都没做。

最近一件事让我很无奈，我在支持其他职场的同事接入我们的平台发布事宜，都是新系统。其实新系统是最好锻炼一个工作流程的事情的，先做好技术架构评审，这会有所有相关的人参加的，你至少知道哪个环节是谁负责的，到时候你要什么资源直接就能找到对应的人。你会发现每到一个阶段交付的当天，这个系统的研发才会找人要资源，比如转测试时候就开始找人了，数据库怎么用户名密码啊、apollo配置啊、发布配置的设置怎么搞啊都在当天找人。到了发布前一天开始提数据库申请流程、域名、VIP申请流程，到发布当天还在沟通这些事情，还有不知道找谁运维发布的。我之前就预感会有这种问题发生，我已经提前两天感觉把我的使用手册@所有相关的人了，让他们看看，并且标注了哪些要提前申请，哪些环境变量要提前配好。连会出现的数据库连接授权我列出来必须要提前跟发布运维沟通设置在发布当天还是会出现这个错误，研发连发布人员是谁也是最后找到我这边去沟通。

可惜是发了，但没人在乎，发布当天我列表里罗列出来的事情被我一一预言命中，让你还能说什么呢，可能还是那句话：人家已经感受到幸福感了，你自己的幸福感值太高别硬拉高别人的幸福感值。卷自己就好了，卷别人不厚道。

### 认知升级

以一个简单的标准，可以把人的认知划分为四个层次：

第一层，不知道自己不知道——以为自己无所不知，自以为是的认知状态。

第二层，知道自己不知道——对未知领域充满敬畏，看到自己的差距与不足，并准备丰富自己的认知。

第三层，知道自己知道——抓住了事情的规律，提升了自己的认知。

第四层，不知道自己知道——保持空杯心态，认知的最高境界。

但我又解释不来这其中的玄学，我只能以我自己的例子来让大家感受一下，我也有认知升级过。

2019年我进入公司那会负责CAT（一个业务调用链统计项目）迁移到3.0版本的事宜，原本CAT2.0是有6台虚拟机的8核16G，因为3.0版本多了些其他功能想更新换代。研究了一段时间后觉得成熟了可以换了，新版给了3台虚拟机也是8核16G的，切换的那时一下子出现数据消息丢失剧增，又赶紧切换会2.0版本，隔天老板还问了一句昨晚CAT回滚了是吧，让我尴尬的要死。

后来我研究了很久发现是机器配置跟不上这么大是数据量，需要更多的机器才行，确实加到了6台机器后切过来的数据就没丢失了。过一段时间又出现问题，是某个应用的数据量太大，落在这台机器的数据很多都是丢失消息。我们加大了内存后确实也解决了，我们也加入了作者开源的微信群，我提出了这个性能问题，作者问了我们是什么配置的机器我们说8核16G，作者说他们的机器每一台都是100多核的物理机。之后我沉默了（我用沉默这个词已经表达了我的无知），很多技术的使用是有硬性条件做支撑的，再牛逼的技术你用10年前的配置你根本玩不转，跑文件也一样，你用SSD盘和用SATA盘的IO性能是无法比的。后面我们也申请了物理机，也做了很多数据库上的优化，现在基本快一年没出过问题了。

第二个是我们组规划的一个项目，要接入非常大的日志数据做分析，但我们的架构要求是降本，我们组在负责这个项目的时候这个同事做了非常多的优化测试、参数调优，最后还是整体不太理想。因为设想要接入全公司的所有应用日志到ES或其他如CK这种数据库是非常大了，某应用一个小时就好几G的日志，人家要查一天的数据库关键词耗时根本缩短不了。就比如在参数调优上做到了极致，如果某一天业务量数据可能稍微有些变化，关键词变的没有规律了，你之前的参数直接不可用，性能急剧下降，这会用户投诉的话，可能你这一年又白干。

我觉得这个项目遇到的瓶颈其实是降本这个词，跟CAT这个场景一模一样，就假设3台机器跑天下的理念会把我们卡的死死的。我想我们需要重新设计假如这个系统到了什么数据量应该考虑资源独享的方案去解决了，如果用共享资源的方案成本是省了，但用起来是牺牲用户的时间的，以空间换时间的做法才是考虑用户角度最好的解决方案，只要做好数据路由代理，那底层的存储你用什么存储技术产品都可以满足需求。最后你需要拿这个方案去说服你的领导，让他同意这个是长期合理方案。

阿里的产品为什么能接这么多用户？他有对云计算是有性能统计的，到什么量级会开启独立资源这个方案都已经在他们的法案里做了备案。所以我认为认知升级是个玄学，你不知道哪天你就想开了，不知道哪天你又凋谢了，但你有积累一些方法论方法清单的话，你不会**想不开**。

### 提问的艺术

提问的艺术这里我必须再次强调，我的另外的一片文章已经写的很清楚了，建议看到这的同学去再看一遍。还是有很多人提问都不会提，你又不能说重了，也不讨喜。

[链接](../../../../self-cultivation/2021/12/16/self-cultivation.html)

### 做个教练型职业人吧

我的领导就是在练习教练型的职业目标，我不知道我是不是这样动机的人，但我很愿意分享我的心得，上面也说过，我遇到了什么坑我都已经按清单方式罗列出来并且给了解决方案思路，我有时候苦恼为啥别人都不在乎你发了什么，我也释然了，总有人会看进去的。我每次在解答别人的问题都是码了很多字，尽量详细：为什么会这样，怎么解决、如果是我们自身bug就要承认并给出后续优化方式、时间等。很多人其实就看怎么解决即可，但我说的详细一些在有心人那里他们就会是经验，下次遇到了这类的问题可能自己就解决了，也可能他们想项目也会遇到这类的问题，这也是他们的经验方案。

我也希望在我的职业道路中有人能点醒我，我记得我在顺丰的时候，有一个架构师带我，我很有啥问题就问他，后面他也烦了说：你自己不会看啊！后来我基本很少问他了，为什么，因为他这句话点醒了我，有问题要自己寻求答案，你花费别人的时间但对别人没有收益，可能还是个低端问题，自己搜索一下就能解决的，你自己都没有付出总让别人给你解答，就是一宗懒的症状。

说到懒，其实开发对测试的不负责也是一种赖的体现，很多开发都是写完代码自己草率测一下赶紧丢给测试去测，我不知道别的组有没有bug统计率和绩效挂钩的，但是在顺丰和阿里是有的，而且及其严重的事情，我之前还被拎出来过：你有一个bug reopen你就死定了。我虽然不喜欢这类的暴政，但确实会让我对自己的代码及其小心谨慎。你可以要求你的测试出具所有场景的测试用例，你按这个用例开发测试，如果有其他问题那一起承担风险。如果代码总依赖测试去帮你挑出来那最后的bug率会极多，秋后算账时项目平平只能拿bug数来看了，一拉统计表好嘛-就是一路长虹，一年又给你玩完。

### 最后

所以我想你写一些心得文章可能也是会帮到一小部分人的，至少你沉淀了点什么。我的职业动机也没多大，有时候我会开玩笑的跟同事说：疫情期间，我要努力保护这份工，别被卷死。

三十大几了也就慢慢佛系了，大不了回家继承家产（赶海），怎么说饿不死，这就是自信。

**开源技术为什么这么好，可能是因为他想开了吧。**