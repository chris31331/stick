# Synchronous Time Interval Consensus Kernel

STICK is a Distributed Consensus Protocol, like PAXOS and RAFT.

Time is divided into rounds, typically one second long.  All servers maintain their time, using NTP, to an accuracy of tens of milliseconds.

Servers are identified by their static internet IP address.

All messages contain the round number.  Delayed messages are discarded.


## Phase Zero

Nothing happens in the first 100ms of each round.  This slack is used to improve the probability of servers agreeing on the time.


## Phase One

In this phase, one of the servers is the leader.  The choice of leader is deterministic, based which hash(leader address ++ round number) is lowest for the current round.  All servers can calculate the leader's identity.  This is Convergent Hashing.


### Proto Consensus

A ProtoConsensus contains a reference to the preceeding consensus and one or more log entries (which are just changes to application data).

(If the leader has no log entries to add, there is no need for a consensus this round.)

The leader sends the ProtoConsensus to the other servers.
```
{
        "protoconsensus": {
                "round": 5,
                "leader": "1.2.3.4",
                "last_consensus": {"round": 4, "hash":"d86e21cc..."},
                "log": [
                        {"add": {"foo": "bar"}},
                ]
        }
}
```

The other servers reply with a Vote, if they agree that it is valid.
```
{
        "vote": {
                "round": 5,
                "voter": "2.3.4.5"
        }
}
```

## Phase Two

At 700ms, the leader counts the received votes.  If more than 50% of the servers have voted for the ProtoConsensus, then the leader publishes the Consensus.
```
{
        "consensus": {
                "round": 5
                "leader": "1.2.3.4",
                "last_consensus": {"round": 4, "hash":"d86e21cc..."},
                "log": [
                        {"add": {"foo": "bar"}},
                ]
        }
}
```

# Issues 

## One Second Granularity

A round duration of at least one second is required to allow 100ms NTP slack and three 200ms one-way internet trips.

On a LAN, with better NTP and shorter trips, 200ms rounds might be possible.

This granularity makes Chronsensus unsuitable for many, but not all, use cases.


## Missed Consensuses

If a non-leader receives a ProtoConsensus, which references a Consensus which it has missed, it must catch up.

If a leader has missed a consensus, which the majority have received, the ProtoConsensus which it send will contain an stale reference and so will not be voted for by the majority of servers.  There will be no consensus that round.


## Failed Servers

When a server fails, no progress will be made during the rounds in which is should have been leader.

It is expected that there will be many servers.  Individual server failure will only cause a minor degradation to service, with 1/N rounds failing to make progress.


## Clients

Clients send log entries to the next round's leader, for incorporation into the next consensus.

Clients will continue to send log entries to each round's leader until they see a consensus containing the log entry.


## Adding and Removing Servers

The address of a server to be added or removed can be contained in a ProtoConsensus's log.  A leader will only incorporate a membership change into a ProtoConsensus when it does not change the order of leaders in the near future.

