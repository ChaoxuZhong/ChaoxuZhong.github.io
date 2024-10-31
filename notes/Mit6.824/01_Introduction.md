# MIT 6.824 分布式系统课程笔记 (Distributed Systems Course Notes)

## 1. 分布式系统的定义与概念
分布式系统是由大量通过网络连接的计算机组成的系统。其关键特征包括：多机器、网络连接、协同工作的计算机系统。

*Definition of distributed systems: a large collection of computers connected by networks. Keywords: multiple, network, cooperating computers - a multiple (more than) one computer networked (so they can interact only through sending or receiving packets).*

## 2. 分布式系统的重要性
分布式系统重要的原因：
- 连接物理隔离的机器，实现数据共享（如用户数据）和协作
- 通过并行化提升系统容量
- 容错能力的提升
- 通过隔离实现安全性

*Why distributed systems are interesting:*
- *Connect physically separated machines, so they can share data like users, enable all kinds of collaborative possibilities*
- *Increase capacity through parallelism*
- *Tolerate faults*
- *Achieve security by isolating them*

## 3. 历史发展脉络
- 1980年代：局域网兴起（如DNS、电子邮件）
- 1990年代：数据中心、大型网站出现
- 2000年代：云计算兴起，满足大规模计算和存储需求，成为公共服务，很多公司开始将基础设施外包给云服务商
- 现今持续发展中

*Historical context:*
- *From local area networks (1980s) - Ex: DNS, email*
- *Data centers, Big websites (1990s)*
- *Cloud computing (2000s)*
- *Current State*

## 4. 技术挑战
分布式系统面临的主要挑战：
- 需要处理大量并发组件
- 必须应对部分故障的情况
- 实现性能优化困难

*Challenges - why distributed systems are so hard:*
- *Many concurrent parts*
- *Must deal with partial failure*
- *Tricky to realize the performance benefits*

## 5. 学习课程的意义
选择学习6.824课程的原因：
- 解决具有挑战性但实用的问题
- 学习真实世界中使用的强大解决方案
- 是活跃的研究领域
- 提供独特的编程实践经验

*Why take 6.824:*
- *Interesting, hard problems but powerful solutions*
- *Used in real world*
- *Active area of research*
- *Hands-on unique style of programming*

## 6. 课程结构
课程包含以下部分：
- 讲座：介绍核心概念
- 论文研究：案例分析
- 编程实验：
   - MapReduce实现
   - 使用Raft的复制系统
   - 复制型KV服务
   - 分片KV服务
     注意：实验包含严格的测试用例，调试困难，需要细心跟踪问题

*Course structure:*
- *Lectures: big ideas*
- *Driving by papers: case study*
- *Programming labs:*
   - *MapReduce*
   - *Replication using Raft*
   - *Replicated KV service*
   - *Sharded KV Service*

## 7.Focos
- infrastructure， not concerned too much with the application at all,we're going to be mostly concerned with infrastructure that supports the applications
- infrastructures falls out three different categories
- 1.storage infrastructur,like key value servers, file systems
- 2.computation like mapreduce
- 3.thrid category is communication , 这会重点在6.829中出现
- what kind of semantics does actually the RPC system provide, at most once exactly once, at lease once

## 8.main topics
- 1.Fault tolerance, availability, recoverablity
- key technique for availability is going to be replication
- key technique for recoverability is logging/ transactions/ durable storage
- 2. consistency, what's consistency basically the ideal is the same behavior as that a single machine would deliver
- 3. performance: different types of consistency or Fault tolerance that's directly related to performance
- one aspect of performance through put,toher is low low latency
- 4. implementation 

## 9.Map reduce
It's a good Illustration of all the problem we will sovle, fault tolerace and performance and tail latency
paper is also very Influential
It's about Lab1
Context and Motivation：背景，google的工程师需要处理大量数据来建立搜索的倒排索引，这样就会用到很多机器来做计算，在长达数小时的执行中，总有计算机会出现问题，所以这本身就是一个分布式系统的容错问题了。 最后的目标是创建 一个库，让非专家也能轻松的使用，这是这篇论文的motivation.
Approach:they take is not a general purpose library,can'e take any application use mapreduce to actually make it basically fault tolerace. mapreduce, functional,stateless.
programmer writes the sequential code, and then hence thess two functions, you know the map and the reduce function to sort of the  framework and the mapreduce gramework deals with all the distributed case

### 9.1.Abstarct view What's going on
1. a bunch of inputs files,basically every file is processed by this map funciton, these map functions all run in parallel,completely dependent with each other, no communication between them, so this is going to give us hopefully high troughput






























