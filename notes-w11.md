
### Unreliable Clocks

- ### Monotonic Versus Time-of-Day Clocks

    - Modern computers have at least two different kinds of clocks: a time-of-day clock and a monotonic clock 

    - Time-of-day clocks 就是我们平常直觉的clock, will return the number of seconds (or milliseconds) since the epoch: midnight
UTC on January 1, 1970, according to the Gregorian calendar. It needs to synchronization according to an NTP server or other external time source in order to be useful. But methods for getting a clock to tell the correct timeare are unreliable:

        -  Quartz clock in a computer is not very accurate: it drifts (runs faster or
slower than it should). Clock drift varies depending on the temperature of the
machine

        - If your NTP daemon is misconfigured, or a firewall is blocking NTP traffic, or large network
delays can cause the NTP client to give up entirely
the clock error due to drift can quickly become large.

        - Leap seconds Problem, result in a minute that is 59 seconds or 61 seconds long, The fact that leap seconds have crashed many large systems

        - In virtual machines, When a CPU
core is shared between virtual machines, each VM is paused for tens of milliseconds while another VM is running. 

        -  you probably cannot trust the device’s hardware clock at all. Some users deliberately set their hardware clock to an incorrect date and time,
for example to circumvent timing limitations in games. 

        In order to help debug market anomalies
such as “flash crashes” and to help detect market manipulation, financial institutions requires all high-frequency trading funds to synchronize their
clocks to within 100 microseconds of UTC

    - A monotonic clock is suitable for measuring a duration (time interval), such as a
timeout or a service’s response time. 不同机器的monotonic clock比较是h没有意义的 it might be the number of nanoseconds since the
computer was started, or something similarly arbitrary. In a distributed system, using a monotonic clock for measuring elapsed time (e.g.,timeouts) because it doesn’t need any synchronization between different nodes’ clocks and is not sensitive to slight inaccuracies of measurement

- ### Relying on Synchronized Clocks

    - software must be
designed on the assumption that the network will occasionally be faulty, and the soft‐
ware must handle such faults gracefully. **The same is true with clocks, robust software needs to be prepared to deal with
incorrect clocks**

    - if you use software that requires synchronized clocks, it is essential that you
also carefully monitor the clock offsets between all the machines. Any node whose
clock drifts too far from the others should be declared dead and removed from the
cluster, Such monitoring ensures that you notice the broken clocks before they can
cause too much damage.

    - ### Timestamps for ordering events

        -  Similar to the one we saw in “Consistent Prefix Reads”
on page 165: the update depends on the prior insert, so we need to make sure that all
nodes process the insert first, and then the update. Simply attaching a timestamp to every write is not sufficient, because clocks cannot be trusted to be sufficiently in sync
to correctly order these events at leader 2.  Figure 8-3 illustrates a dangerous use of time-of-day clocks in a database with multi-leader replication

        - Figure 8-3 展示了，在LWW模式下由于不同机器的 time-of-day clock出现不完全的同步，导致发生在node2丢失了后发生的操作（被更早前发生的操作覆盖了） **Database writes can mysteriously disappear: a node with a lagging clock is unable
to overwrite values previously written by a node with a fast clock.** Some
implementations generate timestamps on the client rather than the server, but this doesn’t change the fundamental problems with LWW

        - 虽然LWW被广泛使用 (LWW is widely used
in both multi-leader replication and leaderless databases such as Cassandra & Riak），但Additional causality tracking mechanisms, such as version vectors, are needed in order to prevent violations of causality

        - Thus, even though it is tempting to resolve conflicts by keeping the most “recent”
value and discarding others, it’s important to be aware that the definition of “recent”
depends on a local time-of-day clock, which may well be incorrect. Even with tightly
NTP-synchronized clocks, you could send a packet at timestamp 100 ms (according
to the sender’s clock) and have it arrive at timestamp 99 ms (according to the recipi‐
ent’s clock)—so it appears as though the packet arrived before it was sent, which is
impossible.

        - 既然NTP受到网络影响，石英钟也不准，那怎么办？So-called logical clocks, which are based on incrementing counters rather
than an oscillating quartz crystal, are a safer alternative for ordering events






    

        

