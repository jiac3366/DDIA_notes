## Chapter 10 The Trouble with Distributed Systems

This chapter is a thoroughly pessimistic and depressing overview of things that may
go wrong in a distributed system

### Unreliable Networks

- ### Faults and Partial Failures

    - 单机只有好和down两种状态，但分布式系统存在partial failure. This nondeterminism and possibility of partial failures is what makes distributed systems hard to work with

    - ### Cloud Computing and Supercomputing

        - a supercomputer is more like a single-node
computer than a distributed system: it deals with partial failure by letting it escalate
into total failure—if any part of the system fails, just let everything crash (like a kernel
panic on a single machine But In this book we focus on systems for implementing internet services, which usually
look very different from supercomputers:

        - If we want to make distributed systems work, we must accept the possibility of partial
failure and build fault-tolerance mechanisms into the softwar

- ### Unreliable Networks

    - If you send a request to another node
and don’t receive a response, it is impossible to tell why

    - ### Network Faults in Practice

        - 即便是系统的网络很稳定， It may make sense to deliberately trigger network problems and
test the system’s response (this is the idea behind Chaos Monkey; see “Reliability” on
page 6).

    - ### Detecting Faults
        
        因为网络的存在，不确定节点是否还在working

        - 如果进程crash了，os会将tcp连接关闭（发送RST FIN包）；如果节点crash了，那么无法得知节点处理了多少数据

        - If a node process crashed (or was killed by an administrator) but the node’s oper‐
ating system is still running, a script can notify other nodes about the crash so
that another node can take over quickly without having to wait for a timeout to
expire. For example, HBase does this

        Rapid feedback about a remote node being down is useful, but you can’t count on it.
Even if TCP acknowledges that a packet was delivered, the application may have
crashed before handling it？？快速反馈远程节点down是有用的，但有时必须假设我们连错误response都收不到


    - ### Timeouts and Unbounded Delays

        - 如果timeout时间过短，则可能是因为节点过载。这时候timout反而造成集群更加过载。When a node is declared dead, its responsibilities need to be transferred to other
nodes, which places additional load on other nodes and the network. transferring its load to other nodes can cause a
cascading failure (in the extreme case, all nodes declare each other dead, and every‐
thing stops working).

        - most systems we work with have neither of those guarantees: asyn‐
chronous networks have unbounded delays (CAN NOT guarantee that every successful request receives a response within time 2d + r) 

        - Timeout时间的确定是动态调整的， automatically adjust time‐
outs according to the observed response time distribution. This can be done with a
Phi Accrual failure detector, which is used for example in Akka and Cassandra. TCP retransmission timeouts also work similarly


        - #### Network congestion and queueing

            网络包延迟现象由于各种排队the variability of packet delays on computer networks is most often
due to queueing

            - If there is so much incoming data that the switch queue fills up, the
packet is dropped, so it needs to be resent—even though the network is function‐
ing fine
            - if all CPU cores are currently
busy, the incoming request from the network is queued by the operating system
until the application is ready to handle it

            - In virtualized environments, a running operating system is often paused for tens
of milliseconds while another virtual machine uses a CPU core. During this time,
the VM cannot consume any data from the network, so the incoming data is
queued (buffered) by the virtual machine monitor, further increasing the
variability of network delays

            - TCP performs flow control （UDP does not perform flow control and does not retransmit lost packets,
it avoids some of the reasons for variable network delays）page 284


    - ### Synchronous Versus Asynchronous Networks

        - compare datacenter networks to the tradi‐
tional fixed-line telephone network(**circuit-switched networks**) (non-cellular, non-VoIP)，电话网络预先为线路建立了固定的带宽？延迟是固定的（bounded delay）为什么不能为互联网tcp网络建立同样的机制？

        -  Why do datacenter networks and the internet use **packet-switching networks** ? The answer is
that they are optimized for bursty traffic. A circuit is good for an audio or video call,
which needs to transfer a fairly constant number of bits per second for the duration
of the call. On the other hand, requesting a web page, sending an email, or transfer‐
ring a file doesn’t have any particular bandwidth requirement—we just want it to
complete as quickly as possible 电话视频的宽带及时性要求比请求网页和发邮件要高得多。Moreover, using circuits for bursty data transfers wastes network capacity and
makes transfers unnecessarily slow 使用电路网络要猜本次传输文件需要多少带宽，猜多浪费猜少又很慢 By contrast, TCP dynamically adapts the rate of
data transfer to the available network capacity.而TCP网络能动态调整

        - With careful use of quality of service (QoS, prioritization and scheduling of
packets) and admission control (rate-limiting senders), it is possible to emulate circuit
switching on packet networks, or provide statistically bounded delay 服务质量和准入控制

        - More generally, you can think of variable delays as a consequence of dynamic
resource partitioning 网络带宽资源和CPU资源的原理类似，异步的思想能提高资源利用率，多租户（multi-tenancy）能够提高资源利用率，但是延迟不变得不确定 but it has the downside of variable delays. 所以延迟不确定并不是网络的必然结果，实际是利益的权衡。we have to assume that network congestion, queueing, and unbounded
delays will happen. Consequently, there’s no “correct” value for timeouts—they need
to be determined experimentally


### Unreliable Clocks

    - 

        

