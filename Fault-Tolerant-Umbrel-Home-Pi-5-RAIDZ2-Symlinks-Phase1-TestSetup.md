# Fault-Tolerant Umbrel Home / Pi 5 with RAIDZ2 symlinks: Phase 1 - Test Setup

Greetings fellow nodesters. [Zap-O-Matic](https://amboss.space/node/026d0169e8c220d8e789de1e7543f84b9041bbb3e819ab14b9824d37caa94f1eb2) here with more node running shenanigans.

> Note: after batch opening 17 channels recently, we have a lot of outbound and are in the process of redistributing. Please connect to us and utilize some of our L-BTC PeerSwap capacity. We will pay the `330 sats` to `swap in` half of the channel balance if you do and you can use that L-BTC to swap out with some of the other amazing [PeerSwap enabled nodes](https://amboss.space/community/181aad24-1871-4be9-ab6d-10ac71383c42) :)
> Not using PeerSwap yet? Read up and join us, it's amazing.
>
> - [PeerSwap Setup Guide](https://stacker.news/items/380494/r/zapomatic)
> - [PeerSwap Economics: Deep Dive Comparing LOOP and the problems with Rebalancing](https://stacker.news/items/382449/r/zapomatic)

Today we have kicked off the first step in testing a theory and would like to share the journey to get more eyes on the setup and process so the community can follow along and help raise awareness to anything absent in the tests and design.

## The Problem

In order to run a Lightning routing node, an operator needs to have a certain amount of DevOps savvy. You have to have solid grasp on networking, power backups, internet backups, fault points within hardware systems, linux command line, etc. But the market is pushing lightning node operation on people who are not yet ready to master these skills (nay, they have only begun to dabble in them).

Solutions like the Umbrel OS, which allow setting up a lightning routing node seemingly out of the box with simple hardware, also allow installing a massive array of peripheral applications that will cause security concerns and performance bottlenecks for a routing node, and the ease of installing the OS on a raspberry pi belies the hardware issues that will plague a serious node. Many new entrants into this hobby are finding that out the hard way. Seeing many people fail to run nodes on raspberry pi hardware with Umbrel, the team created a beefier hardware option called the Umbrel Home, which offers higher ram, NMVe storage, better CPU and cooling, etc.

This is a good start to helping the market be more plug and play ready to create a home node (private or public), but again, it masks over one major problem that every computing system faces: Hard drives fail. Always. Some sad idle Tuesday when you least expect it, it will simply die.
And this points to the major flaw in the Umbrel Home. It has a single built-in 2TB NVMe drive. It sounds great for a home media server where you don't really care about losing data (you can always rebuilt it), but if you want to run your own personal bitcoin bank, this is not nearly good enough.

You might think you can get away with just buying a new Umbrel Home every year or two and powering down and migrating and front-running your hardware failure, but hardware failure is a [Poisson distribution](https://en.wikipedia.org/wiki/Poisson_distribution), not a guaranteed date after purchase. While an NVMe might last 10 years on average, your hardware could fail 6 months after setup. Users need to have storage redundancy and the ability to swap out drives that go bad.
RAID systems are great for disaster mitigation, but they are not without their own flaws. The bigger the storage system, the longer it takes to rebuild failed drives. The more reads and writes and the more drives, the more frequent failures will happen. By adding RAID, you are increasing the failure rate with the caveat that you can now do something to fix it. But managing RAID systems is, again, not something the average person can do.

## Failure Example

One of our channel partners recently had some storage issues where the node ran low on disk space and went offline. When we contacted about it over telegram, the node runner started removing apps and brought the node back online. However, when the node came back, it reconnected and sent us an old HTLC state, which showed up in our logs like so:

```
sync failed: remote believes our tail height is 561, while we have 805!
```

Not messing around with potential malicious states, LND continued the thread:

```
...error when syncing channel states: possible remote commitment state data loss
...failing link: unable to synchronize channel states: possible remote commitment state data loss with ...error: sync error
...Removing channel link with ChannelID(...)
...Force closing link(...)
```

Ouch. Arguably, this is a problem with lightning node logic and a failure to utilize watchtowers. This situation should have been recoverable with a sequence like so:

1. Node A comes back online and sends old state.
2. Node B says, "wait, this is an old state, here's my most recent cryptographically signed HTLC between us, do you accept?"
3. Node A checks with watchtower partners and gets a confirmation that they are out of date and accepts the latest state.
4. Everyone happily moves on with the latest state without force closure.

OR, even better:

1. Node A comes back online and checks with watchtower partners for records of newer state.
2. Watchtower sends back latest state.
3. Node A discovers that their DB is bad/old and recovers from watchtower data.
4. Node A connects to Node B and everything is fine without Node B even knowing that the Node A db was corrupted.

But since neither of these logical flows exist today in Lightning, we have to focus on the problem we can solve, which is that Node A should have storage failover.

## The Proposal

Rather than putting the entire OS and filesystem on RAID, we propose to create a small ZRAID2 pool connected to the system that is responsible for storing ONLY mission critical files, which are then symlinked back to the OS. The goal will be to end up with a plug and play hardware addition to a Pi 5 or Umbrel Home along with scripts for setup and management of ZFS RAID that node runners can use to strengthen their home node fault tolerance.

## The Hardware

The initial hardware tests will include the following:

1. An [Umbrel Home](https://umbrel.com/umbrel-home)
2. An [8GB Raspberry Pi 5](https://www.canakit.com/raspberry-pi-5-8gb.html)
3. 10 [256GB SSD USB Drives](https://www.amazon.com/gp/product/B0C3B18PKT)
4. A [powered USB HUB](https://www.amazon.com/gp/product/B0797NZFYP)
5. [CyberPower Sinewave UPS](https://www.amazon.com/gp/product/B00429N19W/)

## The Test

Once the hardware is delivered and assembled, we will begin testing by creating a node on each hardware device using `testnet`. Each node will install minimal Lightning routing node tools and we will document the setup process along the way, which will include creating scripts to setup and manage the external drive ZRAID pool. We will then run fake traffic on the nodes to create a high db write environment and simulate drive failures using the power buttons on the USB HUB, replacing a "dead" drive with a fresh pool-ready SSD and initiating recovery. If all goes well, the nodes will continue running without the need for SCB or accidental broadcast of out-of-date states, which otherwise triggers a force closure.
