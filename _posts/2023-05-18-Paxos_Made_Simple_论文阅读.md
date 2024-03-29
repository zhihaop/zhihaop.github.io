---
layout: post
title:  "Paxos Made Simple 论文阅读"
date:   2023-05-18 21:01:00 +0800
categories: jekyll update
---
## 共识算法

### 问题

假设一些进程可以提案一些值。共识算法保证在这些提案的值中，只有一个被选中。如果没有值被提案，那么不会有值被选中。如果一个值被选中，进程应该能去学到选中的值。共识算法必须保证以下特性（称为共识算法的安全性）：

1. 只有被提出的值才可能会被选中
2. 只有一个值会被选中，且
3. 一个进程不会获知没有选中的值，除非这个值被选中

### 目标

保证一个提案值被最终选中。如果一个值被选中，所有进程最终（无时间要求）可以学习这个值。

### Paxos 的假设

#### 三个角色

一个进程可能会扮演一个或多个角色

- Proposer：提出提案
- Acceptor：参与决策，接受 Proposer 的提案
- Learner：不参与决策，从 Proposer 和 Acceptor 学习最新达成一致的提案

![image.png](https://raw.githubusercontent.com/zhihaop/zhihaop.github.io/master/_imgs/2023-05-18/0.png)

#### 异步非拜占庭网络

角色间通过网络进行通信，有以下特点

- 角色以任意速度运行，可能宕机或重启（要求有持久化）
- 消息传输可能花任意长的时间，可能会丢失或重复，但是不会被篡改（非拜占庭）
- 消息的传递是异步的（意味着消息可能会乱序到达）

## 如何选出一个值

### 推导过程

#### 使用多个 Acceptors

- 单个 Acceptor，不能容错
- 一个 Proposer 发送一个提案值到多个 Acceptors
- 一个 Acceptor 可能会接受提案值

#### 一个提案值被选中，仅当大多数（半数以上） Acceptors 接受它

任意两个大多数 Acceptors 必定一个共同的 Acceptor。在一个 Acceptor 只接受一个值的情况下，一旦大多数已经接受了一个值，它就不会接受别的值，因此别的值不可能被选中，这保证了值的安全性。如果什么错误都没发生，我们希望有且只有一个值被选中。为了系统的活性，我们提出了 P1。
> Majority（大多数）机制，保证了 Paxos 可以运行在**允许宕机故障**的异步系统中，并保证了2F+1的容错能力，即在2F+1个节点中，最多允许F个节点同时发生故障。

#### P1. Acceptor 必须接受它收到的第一个提案

在 P1 下，多个值可能被多个 Proposer 同时提出，导致没有值被大多数 Acceptor 接受。如果一个 Acceptor 只能接受一个提案，算法就不可行。  

因此我们必须允许 Acceptor 接受多个提案。为了使这些提案区分开来，我们使用一个自然数区分不同的提案，一个提案由编号和值组成，不同的提案具有不同的值。  

一个值被选中当且仅当具有该值的提案被大多数接受，这种情况下我们说一个提案被选中。由于我们允许多个提案被选中，因此必须保证所有选中的提案都有相同的值。因此提出了 P2。

#### P2. 如果一个值$v$的提案被选中，那么任何被选中的、编号较高的提案值必须也是$v$

由于编号单调递增，P2 保证了被选中值的安全性。一个值要被选中，其提案要被至少一个 Acceptor 接受。P2(a) 可以满足 P2。

#### P2(a). 如果一个值$v$的提案被选中，那么每个编号更高的被 Acceptor 接受，其值必须也是$v$

如果没有网络丢包和乱序，P2(a) 是没有问题的。Acceptor 在 P1 约束下，接受第一个提案，并在 P2(a) 约束下，接受编号更大，且值相同的提案。编号更小的提案不能被接受，这必然会违背 P2(a)。  

P2(a)不满足异步网络的假设：由于网络是异步的，即使一个提案被选中了，但是可能存在 Acceptor$a$没有收到任何提案。如果现在有一个新的 Proposer 出现，发起了一个编号更高的提案，并且这个提案的值不是$v$，那么$a$就会接受这个提案，就会违背 P2(a)。  

因此，为了满足P2，必须增强 P2(a)，我们提出 P2(b)。

#### P2(b). 如果提案$\{n,v\}$被选中，则之后任何 Proposer 发出的编号更高的提案值都是$v$

由于提案一定是由 Proposer 发出，被 Acceptor 接受，因此 P2(b) 成立就可以满足 P2(a)，进而满足 P2。为了推导出使 P2(b) 成立需要满足的条件，我们证明 P2(b) 怎样才能成立。  

**NOTE：** 作者在这里使用了第二数学归纳法，它的过程如下：设$P(n)$是一个与自然数有关的命题

1. 证明$n=0$时命题成立
2. 证明当$k < n$成立时，命题$P(n)$也成立
3. 那么，命题$P(n)$对于一切正整数都成立。

**推导：** 假设提案$\{m, v\}$被选中了，命题$P(n)$代表提案$n$具有值$v$。需要证明$n > m$时命题成立

1. 提案$m$具有值$v$，也就是$P(m)$成立。
2. 假设$m \le k < n$的提案都有值$v$，尝试证明$P(n)$成立。
   1. 由于提案$m$被选中，一定存在一个大多数集合$C$接受了提案$\{m,v\}$。
   2. 该假设预示着，每一个$C$中的 Acceptor 接受了提案$\{k,v\}$。
3. 由于多数派集合$S$和$C$必定有一个交集。我们可以得出结论：如果下面 P2(c) 的不变性得到保证，那么命题 $P(n)$成立。

#### P2(c). 对于所有$v$和$n$，如果提案$\{n, v\}$能被提出，必然存在一个多数派集合$S$，满足

1. $S$中没有 Acceptor 接受过编号小于$n$的提案，要么
2. 对于所有$S$中被 所有Acceptor 接受的编号小于$n$的提案中，编号最高的提案值是$v$

我们可以通过维护 P2(c) 不变式来保证 P2(b)，从而保证 P2。
**解释：** 我们对上面两种条件进行解释

1. 对于第一种条件：两种情况，没有提案被选中 或 有提案$m$被选中但$m\ge n$。
2. 对于第二种条件：一种情况，存在提案 $\{m, v\}$，且$m < n$被选中。如果$v$不是最高编号的提案值，提案$\{n,v\}$被提出，那么可能存在一个编号小于$n$的提案$m$，这个提案$\{m,v'\}$被选中，根据 P2(b) 可以得出$v=v'$，与假设不符，违反 P2(b)。

为了去维护 P2(b)，Proposer 在发起提案$n$之前，必须学习提案号小于$n$且编号最大的提案，这个提案**已经**或**将要**被一个多数派的所有 Acceptor 接受。
去知道提案是否已经被接受是容易的，但是预测它未来是否被接受是困难的。对于编号$n$，与其去预测编号小于$n$的提案未来是否会被接受，Proposer 通过一个简单的协议，要求 Acceptor 不再接受编号小于$n$的提案。这就引出了提案发起算法，它通过满足 P2(c) 来满足 P2。

### 提案发起算法

提案发起算法描述了 Proposer 如何发起一个提案，它分为两个部分。

1. **Prepare 请求：** Proposer 选择一个新的编号$n$，将它发送给某些 Acceptor，并要求返回：
   1. 一个承诺：永远不去接受编号小于$n$的协议
   2. 提案：在 Acceptor 接受的编号小于$n$的提案中，编号最大的提案。提案不存在就不回复。
2. **Accept 请求：** 如果 Proposer 收到了大多数 Acceptors 的回复，那么它可以发起提案$\{n,v\}$。如果 Acceptor 没有回复提案，那么$v$的选取是任意的；反之，$v$只能是 Acceptor 回复的提案的提案值。发起提案时，Proposer 向一组 Acceptors （可以和 Prepare 不同）发送提案请求$\{n,v\}$，从而发起提案。

### 提案接受算法

提案接受算法描述了 Acceptor 如何处理 Prepare 请求和 Accept 请求。接受者可以忽略任何请求而不影响安全性，因此我们只需要考虑什么时候，Acceptor 可以对请求进行响应。

#### P1(a). Acceptor 可以接受编号为$n$的提案，当且仅当它没有对编号大于$n$的 Prepare 请求作出回应

观察到 P1(a) 包含 P1，提案发起算法满足 P2，实际上我们已经有一个完备的算法，使得选择一个值满足共识的安全性了。相当于最终的版本，只差了一些小的优化。  

**优化：** 假设一个 Acceptor 收到了一个 Prepare(n) 请求，但是它已经回复了 Prepare(m) 请求（其中$m>n$)，那么 Acceptor 应该忽略这个 Prepare 请求。对于已经接受的提案$n$，Acceptor也应该忽略这个 Prepare(n) 请求。  

**持久化：** 通过这种优化，Acceptor 只需要记住**最高编号的提案** 和 **已响应的 Prepare 请求的最高编号**。这是由于要保证 P2(c) 的不变性，即使 Acceptor 宕机也需要保持上面的信息。  

**注意：** Proposer 总是可以放弃一个提案，并完全忘记它，只要它不发出具有相同编号的提案。

### Paxos 算法

将 Proposer 和 Acceptor 的动作放在一起，我们可以将算法分成两个阶段。

#### Phase 1

1. Proposer 选择一个编号$n$，发送 Prepare(n) 请求到 Acceptor 的多数派
2. 如果 Acceptor 收到了 Prepare(n) 请求，而且$n$大于 Prepare 请求响应的最大编号，那么它就会响应 Proposer，承诺不再接受任何编号小于$n$的提案，并且提供已接受的最高编号的提案。

#### Phase 2

1. 如果 Proposer 接受来自 Acceptor 多数派的 Prepare(n) 响应，它会发起 Accept(n, v) 请求给 Acceptor 多数派。其中，如果 Prepare(n) 响应没有包含提案，那么$v$可以为任意值；反之，$v$只能是 Prepare(n) 响应中的提案值。
2. 如果 Acceptor 接受到 Accept(n, v) 请求，它将接受这个提案，除非它已经对编号大于$n$的 Prepare 请求作出回应。

#### 持久化

Paxos 的正确性依赖于持久化。它需要持久化以下值：

- **Acceptor：** 最高编号的提案，已响应的 Prepare 请求的最高编号
- **Proposer：** 不需要持久化值。但通常为了性能考虑，会持久化最近一次提案的 Proposal ID
- **Learner：** 不需要持久化值。为性能考虑可以持久化选中的提案

> Phase 1 通常又叫 Prepare 阶段，Phase 2 称为 Accept 阶段。
> 在 Prepare 阶段中，主要工作就是选择一个 Proposal ID，并发送一个 Prepare 请求。Acceptor 收到 Prepare 响应后，做出**两个承诺**和**一个应答**。
> 
> 两个承诺是指：
>
> - 不再接受 Proposal ID <= 当前请求的 Prepare 请求
> - 不再接受 Proposal ID < 当前请求的 Propose 请求
> 
> 一个应答是指：
>
> - 已接受的最高编号的提案 或 一个空值  
>
> 在 Accept 阶段中，主要工作就是：Proposer 接受半数以上 Acceptor 的响应，根据 Prepare 阶段的应答结果发起提案；Acceptor 履行承诺，并尝试接受 Proposer 发来的提案。
>
> 有博客将 Learner 学习提案的过程称为 Phase 3，也称为 Learner 阶段。具体的过程在后面给出。

**性能优化**：如果某个 Proposer 开启发起一个更高编号的提案，那么当前 Proposer 放弃提案是个好选择。因此，如果一个 Acceptor 因为收到了更高编号的 Prepare 请求，而放弃当前的 Prepare 或 Accept 请求，那么它应该通知 Proposer 放弃提案。
> **拓展：** 在 Raft 中，Acceptor 让 Proposer 放弃提案是通过 voteGranted 字段和 Term 字段
>
> - 前者告知 Candidate 不能被接受为 Leader
> - 后者决定了一个提案（选举，日志提交）是否能被达成
> 
> **提问：** 那 Raft 中的 success 字段也是上述作用吗
>
> - Term 字段才是真正决定日志能否被接受的，success 字段是用于 Follower 日志追赶（Catchup）

![image.png](https://raw.githubusercontent.com/zhihaop/zhihaop.github.io/master/_imgs/2023-05-18/1.png)


## 学习选中的值

为了去学习选中的值，Learner 必须找到一个被 Acceptor 的多数派接受的提案。最简单的算法就是让每个 Acceptor，只要它接受了提案，就把接受的提案发送给所有 Learner。  

**优点：** Learner 可以尽可能快地学习到选中的提案。  

**缺点：** 每个 Acceptor 都要回应每个 Proposer，网络开销大。

#### 杰出 Learner

非拜占庭失败的假设，使一个 Learner 很容易从另一个 Learner 发现一个值已经被选中。我们可以让 Acceptor 将他们接受的提案发送给一个 **杰出 Learner**，这个 杰出 Learner 在发现一个值被选中时，去通知其他的 Learner。  

**优点：** 是网络开销小  

**缺点：** 杰出的 Learner 可能会 Failure。

#### 杰出 Learner 集合

更具一般性的，可以把上述方法的 杰出 Learner 变成 杰出 Learner 集合。Acceptor 发送提案给 杰出 Learner 集合，Learner 发现值被选中后广播给所有 Learner。

#### Learner 如何判断一个值被选中

由于异步网络的假设，消息可以被丢失。可能出现一个值被选中，但 Learner 永远没有发现的情况。Learner 可以去询问 Acceptor 哪些提案被接受了，但是 Acceptor 的单点失败导致 Learner 不可能知道是否存在提案被 Acceptor 的多数派接受了。  

在这种情况下，当且仅当一个新的提案被选中，Learner 才知道哪个值被选中。在这种情况下，Learner 为了知道一个值是否被选中，可以让 Proposer 发起一个新的提案。（相当于 Raft 的安全性规则：Leader 不能通过复制副本数，提交上一个任期的日志）
> **省流**  
> 
> Learner 学习提案的值，有三种方法：
>
> - 每个 Acceptor 接受提案后都发给所有 Learner，让 Learner 判断提案是否被接受
> - 发给一个 Learner，Learner 判断后广播给其他 Learner
>   - **类似 Raft 日志复制：** 日志复制会捎带 leaderCommit
>   - **类似 Raft Leader 选举：** Leader 选举成功后，会给所有节点发心跳
> - 发给一个 Learner 集合，Learner 集合任一 Learner 判断后，广播给其他 Learner
>
> Learner 怎么主动获知值是否被选中：
> - **主动询问：**有活性问题，可能永远都不会有值被选中
> - **提交提案：**Proposer 发起一个新的提案，Learner 要么知道这个值被选中，要么知道之前的值
>
> **拓展**  
>
> 上面有点像 Raft 的安全性规则：Leader 不能通过复制副本数，提交上一个任期的日志。只能通过提交一个新日志，捎带提交旧任期的数据。
>
> 1. 为什么不能提交旧任期的数据：因为日志复制到大多数节点，不意味这个日志已经被提交
> 2. 为什么不能主动获知旧任期的日志提交状态：因为活性问题，有可能旧任期的日志永远不会被提交
> 3. 如果获知上一个任期的日志提交状态（如何提交旧任期的日志）：提交一个新的日志

## 取得进展

一个算法具有活性，意味着算法最终能取得进展。对于多 Proposer 系统，可以容易构建一个场景，两个 Proposer 发起一系列的编号递增的提案，没有任何一个提案被选中，系统无法取得进展。

1. Proposer p 以编号$n_1$完成了 Phase 1。
2. Proposer q 以编号$n_2>n_1$完成了 Phase 1。
3. Proposer p 发起 Accept 请求，但被拒绝（由于 2 的缘故）
4. Proposer p 以编号$n_3>n_2$完成了 Phase 1。
5. Proposer q 发起 Accept 请求，但被拒绝（由于 4 的缘故）
6. 依次类推，形成活锁。

为了保证算法能取得进展，必须选出一个 杰出 Proposer，它是唯一能发起提案的人。  

如果杰出Proposer 能成功与 Acceptor 的多数派通信，并且它发起的提案编号大于所有用过的编号，那么它将成功发出一个提案，并且该提案能被接受。通过放弃一些提案，并得知有更高的编号时再次重试，杰出 Proposer 最终会选择一个足够高的提案编号。  

我们可以选举一个杰出 Proposer 来实现活性（解决活锁问题）。由于 FLP不可能原理，选举一个 Proposer 必须使用随机算法 或 真实时间（比如使用随机超时）。
> **省流**  
> 
> 总而言之，算法要具有活性，必须满足下面三个要求之一：
>
> - 只有一个提案者（类比 Raft 只有 Leader 能提交日志）
> - 使用随机算法（类比 Raft 的选举随机超时）
> - 使用真实时间（类比 Raft 的选举随机超时）

## 实现

Paxos 算法假设进程间使用网络通信。在这个共识算法中，每个进程扮演着 Proposer，Acceptor 和 Learner 的角色。  

该算法首先选出一个 Leader，它既是杰出的 Proposer，又是杰出的 Learner。通过文中描述的 Paxos 算法，进行提案的发起和接受。稳定存储用于维护 Acceptor 必须记得的信息，防止宕机破坏共识算法安全性。  

为了保证不同的提案具有不同的编号，每个 Proposer 应该从互不相交的自然数集合中选择编号。为了防止宕机破坏这个机制，每个 Proposer 都应该记住它曾发起的最高编号的提案，并且在 Prepare 阶段的编号，应该比用过的最高编号还有大。

### 实现复制状态机（Mutli-Paxos）

#### 中央服务器

一堆客户端，向一个中央服务器发布命令。服务器可以被描述成一个确定性状态机，它以某种顺序执行客户端命令。状态机有一个当前状态，它接受一个命令作为输入，并产生一个输出和当前状态。

#### 服务器集合

上述系统在中央服务器宕机时，会产生单点问题。因此使用服务器集合，每个服务器独立实现状态机，如果每个服务器执行相同的命令序列，就会产生相同状态和输出序列。客户端可以使用任意服务器的输出。  

为保证所有服务器都执行相同的命令序列，可以通过 Paxos 算法的一系列独立实例，其中第$i$个实例选中的值是序列中的第$i$个状态机命令。每个服务器扮演所有的角色（Proposer，Acceptor，Learner）。  

假设服务器的集合是固定的，那么共识算法的所有实例会使用相同的代理人集合。在通常情况下，一个服务器被选举为 Leader，它在所有 Paxos 实例中扮演杰出 Proposer 的角色。客户端向 Leader 发送命令，Leader 决定命令在命令序列中的位置。如果 Leader 认为这条命令应该出现在第$i$个位置，那么它试图让这个指令被选为第$i$个 Paxos 实例的值。这个操作有可能失败，可能另一个服务器也认为自己是 Leader，对第$i$个命令有不同看法。但是 Paxos 算法确保最多只有一条命令可以出现在第$i$个位置。这个方法的有效性在于，在 Paxos 算法中，只有在 Phase 2 才能决定一个值是否被选中。
> **省流**
>
> 1. 有多个服务器，每个服务器既是 Proposer，Acceptor，也是 Learner
> 2. 每个服务器运行多个 Paxos 实例，对应着唯一的 Instance ID。第$i$个值由第$i$个 Paxos 实例决定
> 3. 多个服务器中，有一个服务器被选举为 Leader，它是杰出的 Proposer 和 Learner。
> 4. 杰出的 Proposer 成功提交一个值，杰出 Learner 就可以知道该值被选中，从而把值发送给其他 Learner（这里有点像 Raft 有没有）
> 5. 如果同时存在多个 Leader，第$i$个值的安全性也能保证。因为每个值都需要 Paxos 实例确定

#### 提出多个值

新的 Leader，作为所有 Paxos 实例的 Learner，应该知道大部分已经被选中的值。假设它已经知道命令1-134、138、139被选中了，然后它执行 135-137 和所有大于 139 实例的 Phase 1。假设 Phase 1 的执行结果确定了在实例 135 和 140 中要发起的值，同时在其他实例中值不受约束。那么 Leader 就会执行 135 和 140 实例的 Phase 2，选中命令 135 和 140。  

Leader 和 其他所有从 Leader 学到命令序列的服务器，现在可以执行命令 1-135。但是对于它们都知晓的 138-140 命令，由于 136 和 137 命令没有被选中，因此不能执行。  

Leader 可以从拿两个命令作为 136 和 137。但是，我们建议通过 no-op 命令立即填补 136 和 137 处的空白。只要 no-op 命令被 136 和 137 实例选中，138-140 命令就可以立即执行。
现在 1-140 命令都已经选中了，并且 Leader 已经对于大于 140 的 Paxos 实例完成了 Phase 1，它可以在 Phase 2 在这些实例中提交任意值。它将编号 141 分配给客户端请求的下一个命令，然后执行 Phase 2，然后分配 142 ... ，依次类推。  

Leader 可以在得知 141 命令已经被选中前，可以发起 142 命令的提议。它在 提议 141 命令时的所有消息都可能丢失，并且在其他任何服务器学到 141 命令前 142 命令被选中也是可能的。当 Leader 未能收到 Phase 2 对 141 命令的响应信息时，它将会重试 Phase 2。如果一切顺利，那么 141 命令将被选中。然而，它也可能会失败，那么命令序列将在 141 处空缺。一般来说，若 Leader 可以提前得到$m$个命令，那么在命令$1..i$选中后，Leader 可以提议$i+1..i+m$命令，这个时候就会出现$m$个命令的间隙。  

新上任的 Leader 对无限多的 Paxos 实例执行 Phase 1。对于上面的例子来说，Leader 会对 135-137 和 所有大于 139 的实例执行 Phase 1，对所有的实例使用相同的提案编号。由于使用了相同提案编号，我们可以向其他服务器发送一个消息，实现对无限多的实例执行 Phase 1。
> **省流**
>
> 1. Leader 应该知道绝大部分被选中的值（Raft 的 Leader 必须知道所有已提交的日志）
> 2. 命令序列可以存在空洞（Raft 不允许日志空洞）
> 3. 如果命令序列存在空洞，空洞后的命令不能执行（Parallel Raft 支持乱序 Commit）
> 4. Prepare 阶段会延申到后面的所有实例，所有实例使用相同的 Proposal ID。后续提交时只需单独执行 Accept 阶段。（有点像 Raft 的 Term）

#### Leader 变更

上面的讨论都假定了系统总有一个 Leader，除了 Leader 宕机 与  Leader 当选 之间的短暂时间。在异常情况下，Leader 选举可能会失败。如果没有服务器充当 Leader，那么没有新的命令会被提议。  

如果多个服务器认为它们是 Leader，那么它们都可以在同一个 Paxos 算法实例中对值进行提议，但是这可能会导致活性问题，但是不会导致安全性问题。  

由于 Leader 宕机和 新 Leader 选举发生的概率很小，因此执行一个状态机命令的有效成本（在值上达成共识的成本），是执行 Phase 2 的成本。可以证明，Paxos 共识算法在 Phase 2 允许存在故障的情况下，是任何算法中达成协议的最小成本。因此，Paxos 算法本质上是最优的。
> **省流**
>
> 1. Leader 选举不常发生，多个 Leader 同时运行不会导致安全性问题
> 2. Paxos 算法在本质上是最优的

#### 成员变更

如果服务器集合可以发生改变，那么必须有某种方法知道 服务器 和 Paxos 实例的对应关系。最简单的方法是通过状态机本身，让当前的服务器集合成为状态的一部分，并且可以通过普通的状态机命令去改变。成员变更可能会让 Leader 节点失效，造成性能下降，因此需要延缓变更的生效。我们可以定义一个延缓窗口$w$，成员变更点为$i$，那么生效点就是$i+w$。我们规定旧的 Leader 在$i+w$前仍能正常提案，而新节点必须在$i+w$开始才能参与提案，完成平滑的过渡。
> **省流**
>
> 1. 服务器节点是状态的一部分，并且能通过普通的状态机命令改变
> 2. 不可能同时更改所有服务器节点状态，需要有一个延缓窗口
> 3. 延缓窗口下，成员变更点为$i$，那么生效点是$i+w$，完成平滑过渡
