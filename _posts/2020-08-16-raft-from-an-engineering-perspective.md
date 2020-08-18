---
layout: post
title: "Raft, from an engineering perspective"
date: 2020-08-16 09:33 -0700
---
I recently completed an implementation of the Raft Consensus algorithm! It is part of the homework of the online version of [MIT course 6.824](https://pdos.csail.mit.edu/6.824/).

The algorithm itself is simple and understandable, as promised by the [paper](https://raft.github.io/raft.pdf).
I'd like to share my experience as an engineer implementing it. 

## Raft

The Raft algorithms is useful, in the sense that it allows users to start abitrary "contracts". Once a contract is executed, it will stay in the executed state,
and survive power outages, server reboots and network failures.

In practise, Raft keeps a distributed log between a list of servers. Once a log entry is committed,
it is guaranteed to stay in the committed state. One of the servers is elected as the leader, and the rest are called followers.
The leader is responsible for serving external users, and keeping followers up-to-date about committed logs.
When the leader dies, or is unreachable, a follower can turn into the leader and keep the system running.

## Core States

When implementing Raft, we maintain a list of core states on each server. The states include the current leader, the log entries, the commited ones,
what was the last term and last vote, when is the time to start an election, and other logistic information. On each server, the states are guarded by a global lock.
Two RPC systems are used to communicate and synchronize states between servers.

Looking back at my implementation, I divide Raft into 6 components.


## Election and Voting 

Responsible to elect a leader to run the system. Arguably the most important part of Raft.

An election is triggered by a timer. A follower starts an election, when it has not hear from a leader for some time.
The follower sends one RPC to each of the peers, asking for a vote. If it could collect enough votes before someone
else starts a new term, then it becomes the new leader. The timer will be reset when a follower hears from the leader.

There are a fair amount of asynchronous operation happening. First, If an election is triggered by a timer,
we could have two elections running at the same time. In my implementation, I made an effort to end the prior
election before starting the new one. This helps reducing the noise in the log and simplifies the states that
must be considered.
Second, latency matters in a tough environment. A candidate should count votes ASAP when it received responses
from peers, and a newly-become leader must notify its peers ASAP that it has collected enough votes.
Third, , when the system is shutdown, an election should be ended as well.

## Heartbeats

The leader sends heartbeat to followers, to ensure that followers know the leader is still alive and functioning. Heartbeats keep the system stable. Followers will not attempt to become a leader when they receive heartbeats (or `AppendEntries` ).

The leader also triggers a immediate round of heartbeats after it has won an election, to declare "follow me".

Triggered by the heartbeat timer. Heartbeat timer should expire faster than any followers' election timer. Otherwise those followers will attempt to run a election first.

## Log Entry Syncing
The leader is responsible to keep all followers on the same page, by sending out AppendEntries requests. 

Triggered by events, i.e. a new contract is started. Monitored by a timer, meaning timed-out attempts should be retried.

## Internal RPC serving
Namely answering AppendEntries and RequestVote. Static and procedural, no waiting required to decide what answer to give.

## Command Applying
When a contract is established, the command attached to a contract is executed. All followers should eventually be aware that a contract is executed.

Triggered by event. No RPC is involved. The system is made async because it communicates with external systems which might be arbitrarily slow.

## External RPC serving
Answer calls from external clients that
1. start a new contract
2. checking an established contract
This part is not implemented, because the way the homework is setup.

## Afterthoughts & Comments
1. Latency matters in a distributed system, because it is tied directly to availability. The longer the latency, the longer the time when the system is not available.
2. Channels can appear to be not fast enough. The receiving end might not be awaken immediately, causing delays. This can be critical when there is a small time window, during which an action must be performed, i.e. realize that I have been elected and declare that I have won.
3. RPCs should be retried.
4. Mutex and condition variables are handy.
5. But sometimes I only need a Semaphore, not the locking part.
6. Cond var is not guaranteed?
7. Modular design wins.
8. Disk delay is not emulated.


