---
title: raft
date: 2021-09-22 00:40
---
**强烈建议大家先移步 **[**http://thesecretlivesofdata.com/raft/#home**](http://thesecretlivesofdata.com/raft/#home)** ，花5分钟左右时间简单了解一下raft。**
​本篇文章主要谈一谈我对raft的一些理解，如有异议，欢迎各路大佬指出～
(下面讨论的是论文版Raft，各种改进版在此不做讨论）

# 1 对Raft算法的理解
## 1.0 raft解决了什么问题
相信大家在接触新事物之前都会有此类疑问，一堆为什么要问。不过【raft解决了什么问题】这个问题可以去问问谷老师或者某度哈哈哈。​
## 1.1 概述
raft基于paxos，是一种用来管理复制日志(replicated log)的共识算法，所以也可以理解为，raft的各种设计都是为了让集群中的server在log上达成共识。
raft的作者认为paxos有两个“缺点”：1）paxos非常难理解。2）paxos工业上不好实现。如果非要说难理解是缺点的话倒也算是吧，毕竟大道至简。


一个raft集群有多台server(~~废话，一台server也不需要共识~~)，给定某个时间点，每个server的状态能且只能为下面三种状态之一：

- leader
   - leader诞生制度为选举制 (个人感觉raft准确来说是毛遂自荐+拉票)
   - 一个集群有且仅有一个leader 所有人都听它的
- follower
   - 集群中机器初始化状态为follower
   - 每个follower都有个选举超时时间(election timeout)，可先简单理解为follower在这个时间内没收到leader发来的心跳，则认为集群当前无leader，可以开始竞选leader了
- candidate
   - 每个想要成为leader的follower，都会先从follower变为candidate状态
   - candidate先给自己投一票，如果能够获得集群中大多数投票则成功成为leader

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/326239/1632043817635-5bddfb66-d622-4ab9-ab05-04404acc2157.png#height=206&id=j25Th&margin=%5Bobject%20Object%5D&name=image.png&originHeight=412&originWidth=888&originalType=binary&ratio=1&size=147643&status=done&style=none&width=444)
集群中一个server的状态机如上图。简单起见，后面说的leader大多都是指【状态为leader的server】，follower/candidate同理。
​
正常情况下，集群只会有一个leader，多个follower/candidate。leader负责处理所有client的请求，而follower/candidate只和leader通信。**（一个集群中能否有两个leader？是否可行？两个leader相比一个leader又有什么好处/坏处？）**
历史的车轮滚滚向前，朝代更替，永不停息。raft的世界亦如是，每个leader的诞生都伴随着一次或多次竞选（election），每一次竞选都会有一个或者多个follower变为candidate参加竞选，而每一次竞选的开始都意味着一个term的开始，每个term对应一个term number（此term number只增不减），因此，term number可称之为raft集群中的逻辑时钟(logical clock)。**（为什么要专门搞个term number呢？用时间值不是更准确？万一有人有类似疑问，可以学习 分布式时钟 相关知识）**
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/326239/1632042837508-0ecc2902-eabe-410e-b6dd-08e17517a933.png#height=150&id=Gddfn&margin=%5Bobject%20Object%5D&name=image.png&originHeight=300&originWidth=738&originalType=binary&ratio=1&size=69155&status=done&style=none&width=369)**​**
正如前面说的，一个term开始于一次竞选，而term以何种方式结束取决于竞选结果。如果竞选中没有一个candidate获得大多数的投票，则认为大家都竞选失败，此次term结束，开始下一轮竞选，如上图中t3。如果有candidate成功成为leader，则在该leader不再是leader之前称之为一个term，如上图term1 term2 term4...
​

集群中server之间通过RPC通信，基本的共识算法需要两种类型的RPC：
1）RequestVote RPC. 该类型RPC由candidate在竞选的过程中生成。
2）AppendEntries RPC. 该类型RPC由leader生成，主要用于日志同步和心跳。
**为什么要这两种分开呢？一种RPC可以吗？**


接下来从日志复制、领导者选举、成员变更三个方面进行介绍。raft论文本身是在讲完选主和日志复制后又统一讲了safety的问题，即如何保证集群能够达成共识，但为了达到safety的目的，限制都是加在了选主和日志复制过程里，因此本文打算将safety的内容糅合到选主和日志复制中一并说明。
## 1.1 领导者选举 Leader Election


raft选主为多数原则，即超过半数的server赞成，则选主流程结束，此过程保证只有一个server赢得竞选。为了保证只有一个candidate竞选成功，在集群中节点投票时，**每个节点按照first-come-first-served的原则，只能投出一票**。


发生选举的情况大致为下面几类：

- 集群为初始态，此时所有server均为follower，需要选主。
- leader无法发出心跳，需要选主。
- follower未收到leader发出的心跳，需要选主。



每个follower都有一个选举超时时间(election timeout)，在一个follwer选举超时时间内如果没有收到来自leader/candidate的有效RPC，则该follower认为此时集群没有leader，状态变为candidate，发起一场竞选。
在candidate(记为x)发起竞选时，分别进行下面四步操作：

1. 自增term
1. 给自己投票
1. 重置自己的election timeout
1. 向集群中其他server发出RequestVote RPC

​

此时，candidate x的竞选结果可能有以下几种：

- x竞选成功，成为leader
- x竞选失败，其他candidate为leader
- 没有任何candidate竞选成功

​

下面对candidate x的上述三种结果分别作出解释。
第一种情况，x竞选成功，成为leader。当x获得了集群中大多数节点的投票之后，x成为leader，并开始向集群中其他server发出AppendEntries RPC，保持自己的leader地位。
第二种情况，x竞选失败，其他candidate成为leader。在x等待集群其他server的投票结果时，收到了来自leader y的AppendEntries RPC，此时，
如果y.term>=x.term，证明当前集群无需选主，leader y即为集群leader，x从candidate变为follower。
如果y.term<x.term，则x拒绝该RPC，继续保持candidate状态。
第三种情况，没有任何server竞选成功。原因可能为：
如果 candidate y 和 candidate x 的 election timeout 差不多，而x和y各收到了集群少部分server的投票，没有任何一方满足多数原则，因此无人成功，坐等下一轮竞选。
如果 candidate x 有机会获得 大多数 server 的投票，但因为资格不够被其他server拒绝 (后面会讲这种case)


举例如下：
如下图，假设集群初始化时，所有server端配置为图中给定值。
![raft01.jpeg](https://intranetproxy.alipay.com/skylark/lark/0/2021/jpeg/326239/1624026911244-8a964c75-c84c-41d3-af99-719f59401b6b.jpeg#height=277&id=ud46a4e2d&margin=%5Bobject%20Object%5D&name=raft01.jpeg&originHeight=898&originWidth=1809&originalType=binary&ratio=1&size=182094&status=done&style=none&width=559)
显然，S2的election timeout最小，因此，S2会成为candidate，重置自己的election timeout，并将term number +1，然后投自己一票后向其他server发出RequestVote RPC，如下图：
![IMG_9C2DDA1E0957-1.jpeg](https://intranetproxy.alipay.com/skylark/lark/0/2021/jpeg/326239/1632045863460-37726f57-118a-4316-92c9-3902dd3f944a.jpeg#height=299&id=dOuvQ&margin=%5Bobject%20Object%5D&name=IMG_9C2DDA1E0957-1.jpeg&originHeight=960&originWidth=1785&originalType=binary&ratio=1&size=194863&status=done&style=none&width=556)
如果S2很幸运的得到了大多数server的同意，则成为leader；


上面我们说过，raft是一个为了让集群中的server在log上达成共识的算法，那么，server在给一个candidate投票时，是依据什么来决定投同意还是拒绝？除了判断term number的大小之外，是否需要满足其他的条件？
## 1.2 日志复制
此部分的前提是，假设集群当前已选好leader，






## 1.3 成员变更








# 2 raft有什么不足吗？








# 3 参考链接
[http://thesecretlivesofdata.com/raft/#home](http://thesecretlivesofdata.com/raft/#home)
[https://raft.github.io/](https://raft.github.io/)