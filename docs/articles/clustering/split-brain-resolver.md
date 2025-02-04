---
uid: split-brain-resolver
title: Split Brain Resolver
---
# Split Brain Resolver

When working with an Akka.NET cluster, you must consider how to handle [network partitions](https://en.wikipedia.org/wiki/Network_partition) (a.k.a. split brain scenarios) and machine crashes (including .NET CLR/Core and hardware failures). This is crucial for correct behavior of your cluster, especially if you use Cluster Singleton or Cluster Sharding.

## The Problem

One of the common problems present in distributed systems are potential hardware failures. Things like garbage collection pauses, machine crashes or network partitions happen all the time. Moreover it is impossible to distinguish between them. Different cause can have different result on our cluster. A careful balance here is highly desired:

* From one side we may want to detect crashed nodes as fast as possible and remove them from the cluster.
* However, things like network partitions may be only temporary. For this reason it may be more feasible to wait a while for disconnected nodes in hope, that they will be able to reconnect soon.

Networks partitions also bring different problems - the natural result of such event is a risk of splitting a single cluster into two or more independent ones, unaware of each others existence. This comes with certain risks. Even more, some of the Akka.NET cluster features may be unavailable or malfunctioning in such scenario.

To solve this kind of problems we need to determine a common strategy, in which every node will come to the same deterministic conclusion about which node should live and which one should die, even if it won't be able to communicate with others.

Since Akka.NET cluster is working in peer-to-peer mode, it means that there is no single *global* entity which is able to arbitrarily define one true state of the cluster. Instead each node has so called failure detector, which tracks the responsiveness and checks health of other connected nodes. This allows us to create a *local* node perspective on the overall cluster state.

In the past the only available opt-in strategy was an auto-down, in which each node was automatically downing others after reaching a certain period of unreachability. While this approach was enough to react on machine crashes, it was failing in face of network partitions: if cluster was split into two or more parts due to network connectivity issues, each one of them would simply consider others as down. This would lead to having several independent clusters not knowing about each other. It is especially disastrous in case of Cluster Singleton and Cluster Sharding features, both relying on having only one actor instance living in the cluster at the same time.

Split brain resolver feature brings ability to apply different strategies for managing node lifecycle in face of network issues and machine crashes. It works as a custom downing provider. Therefore in order to use it, **all of your Akka.NET cluster nodes must define it with the same configuration**. Here's how minimal configuration looks like:

```hocon
akka.cluster {
  downing-provider-class = "Akka.Cluster.SBR.SplitBrainResolverProvider, Akka.Cluster"
  split-brain-resolver {
    active-strategy = <your-strategy>
  }
}
```

Keep in mind that split brain resolver will NOT work when `akka.cluster.auto-down-unreachable-after` is used.

## Split Brain Resolution Strategies

Beginning in Akka.NET v1.4.16, the Akka.NET project has ported the original split brain resolver implementations from Lightbend as they are now open source. The following section of documentation describes how Akka.NET's hand-rolled split brain resolvers are implemented.

### Picking a Strategy

In order to enable an Akka.NET split brain resolver in your cluster (they are not enabled by default), you will want to update your `akka.cluster` HOCON configuration to the following:

```hocon
akka.cluster {
  downing-provider-class = "Akka.Cluster.SBR.SplitBrainResolverProvider, Akka.Cluster"
  split-brain-resolver {
    active-strategy = <your-strategy>
  }
}
```

This will cause the [`Akka.Cluster.SBR.SplitBrainResolverProvider`](xref:Akka.Cluster.SBR.SplitBrainResolverProvider) to be loaded by Akka.Cluster at startup and it will automatically begin executing your configured `active-strategy` on each member node that joins the cluster.

> [!IMPORTANT]
> _The cluster leader_ on either side of the partition is the only one who actually downs unreachable nodes in the event of a split brain - so it is _essential_ that every node in the cluster be configured using the same split brain resolver settings. Otherwise there's no way to guarantee predictable behavior when network partitions occur.

The following strategies are supported:

* `static-quorum`
* `keep-majority`
* `keep-oldest`
* `down-all`
* `lease-majority`
* `keep-referee` - only available with the legacy split brain resolver.

All strategies will be applied only after cluster state has reached stability for specified time threshold (no nodes transitioning between different states for some time), specified by `stable-after` setting. Nodes which are joining will not affect this threshold, as they won't be promoted to UP status in face unreachable nodes. For the same reason they won't be taken into account, when a strategy will be applied.

```hocon
akka.cluster.downing-provider-class = "Akka.Cluster.SBR.SplitBrainResolverProvider, Akka.Cluster"
akka.cluster.split-brain-resolver {
  # Enable one of the available strategies (see descriptions below):
  # static-quorum, keep-majority, keep-oldest, keep-referee 
  active-strategy = off
  
  # Decision is taken by the strategy when there has been no membership or
  # reachability changes for this duration, i.e. the cluster state is stable.
  stable-after = 20s

  # When reachability observations by the failure detector are changed the SBR decisions
  # are deferred until there are no changes within the 'stable-after' duration.
  # If this continues for too long it might be an indication of an unstable system/network
  # and it could result in delayed or conflicting decisions on separate sides of a network
  # partition.
  #
  # As a precaution for that scenario all nodes are downed if no decision is made within
  # `stable-after + down-all-when-unstable` from the first unreachability event.
  # The measurement is reset if all unreachable have been healed, downed or removed, or
  # if there are no changes within `stable-after * 2`.
  #
  # The value can be on, off, or a duration.
  #
  # By default it is 'on' and then it is derived to be 3/4 of stable-after, but not less than
  # 4 seconds.
  down-all-when-unstable = on
}   
```

There is no simple way to decide the value of `stable-after`, as:

* A shorter value will give you the faster reaction time for unreachable nodes at cost of higher risk of false positives, i.e. healthy nodes that are slow to be observed as reachable again prematurely being removed for the cluster due to temporary network issues.
* A higher value will increase the amount of time it takes to move resources on the truly unreachable side of the partition, i.e. sharded actors, cluster singletons, DData replicas, and so on longer to be re-homed onto reachable nodes in the healthy partition.

> [!NOTE]
> The rule of thumb for this setting is to set `stable-after` to `log10(maxExpectedNumberOfNodes) * 10`.

The `down-all-when-unstable` option, which is _enabled by default_, will terminate the entire cluster in the event that cluster instability lasts for longer than the `stable-after` + 3/4 of the `stable-after` value in seconds - so by default, 35 seconds.

> [!IMPORTANT]
> If you are running in an environment where processes are not automatically restarted in the event of an unplanned termination (i.e. Kubernetes), we strongly recommend that you disable this setting by setting `akka.cluster.split-brain-resolver.down-all-when-unstable = off`.
> If you're running in a self-hosted environment or on infrastructure as a service, TURN THIS SETTING OFF unless you have automatic process supervision in-place (which you should always try to have.)

#### Static Quorum

The `static-quorum` strategy works well, when you are able to define minimum required cluster size. It will down unreachable nodes if the number of reachable ones is greater than or equal to a configured `quorum-size`. Otherwise reachable ones will be downed.

When to use it? When you have a cluster with fixed size of nodes or fixed size of nodes with specific role.

Things to keep in mind:

1. If cluster will split into more than 2 parts, each one smaller than the `quorum-size`, this strategy may bring down the whole cluster.
2. If the cluster will grow 2 times beyond `quorum-size`, there is still a potential risk of having cluster splitting into two if a network partition will occur.
3. If during cluster initialization some nodes will become unreachable, there is a risk of putting the cluster down - since strategy will apply before cluster will reach quorum size. For this reason it's a good thing to define `akka.cluster.min-nr-of-members` to a higher value than actual `quorum-size`.
4. Don't forget to add new nodes back once some of them were removed.

This strategy can work over a subset of cluster nodes by defining a specific `role`. This is useful when some of your nodes are more important than others and you can prioritize them during quorum check. You can also use it to to configure a "core" set of nodes, while still being free grow your cluster over initial limit. Of course this will leave your cluster more vulnerable in situation where those "core" nodes will fail.

Configuration:

```hocon
akka.cluster.split-brain-resolver {
  active-strategy = static-quorum

  static-quorum {
    # minimum number of nodes that the cluster must have 
    quorum-size = undefined
        
    # if the 'role' is defined the decision is based only on members with that 'role'
    role = ""
  }
}
```

#### Keep Majority

The `keep-majority` strategy will down this part of the cluster, which sees a lesser part of the whole cluster. This choice is made based on the latest known state of the cluster. When cluster will split into two equal parts, the one which contains the lowest address, will survive.

When to use it? When your cluster can grow or shrink very dynamically.

Keep in mind, that:

1. Two parts of the cluster may make their decision based on the different state of the cluster, as it's relative for each node. In practice, the risk of it is quite small.
2. If there are more than 2 partitions, and none of them has reached the majority, the whole cluster may go down.
3. If more than half of the cluster nodes will go down at once, the remaining ones will also down themselves, as they didn't reached the majority (based on the last known cluster state).

Just like in the case of static quorum, you may decide to make decisions based only on a nodes having configured `role`. The advantages here are similar to those of the static quorum.

Configuration:

```hocon
akka.cluster.split-brain-resolver {
  active-strategy = keep-majority

  keep-majority {
    # if the 'role' is defined the decision is based only on members with that 'role'
    role = ""
  }
}
```

#### Keep Oldest

The `keep-oldest` strategy, when a network split has happened, will down a part of the cluster which doesn't contain the oldest node.

When to use it? This approach is particularly good in combination with Cluster Singleton, which usually is running on the oldest cluster member. It's also useful, when you have a one starter node configured as `akka.cluster.seed-nodes` for others, which will still allow you to add and remove members using its address.

Keep in mind, that:

1. When the oldest node will get partitioned from others, it will be downed itself and the next oldest one will pick up its role. This is possible thanks to `down-if-alone` setting.
2. If `down-if-alone` option will be set to `off`, a whole cluster will be dependent on the availability of this single node.
3. There is a risk, that if partition will split cluster into two unequal parts i.e. 2 nodes with the oldest one present and 20 remaining ones, the majority of the cluster will go down.
4. Since the oldest node is determined on the latest known state of the cluster, there is a small risk that during partition, two parts of the cluster will both consider themselves having the oldest member on their side. While this is very rare situation, you still may end up having two independent clusters after split occurrence.

Just like in previous cases, a `role` setting can be used to determine the oldest member across all having specified role.

Configuration:

```hocon
akka.cluster.split-brain-resolver {
  active-strategy = keep-oldest

  keep-oldest {
    # Enable downing of the oldest node when it is partitioned from all other nodes
    down-if-alone = on
    
    # if the 'role' is defined the decision is based only on members with that 'role',
    # i.e. using the oldest member (singleton) within the nodes with that role
    role = ""
  }
}
```

#### Down All

As the name implies, this strategy results in all members of the being downed unconditionally - forcing a full rebuild and recreation of the entire cluster if there are any unreachable nodes alive for longer than  `akka.cluster.split-brain-resolver.stable-after` (20 seconds by default.)

You can enable this strategy via the following:

```hocon
akka.cluster {
  downing-provider-class = "Akka.Cluster.SBR.SplitBrainResolverProvider, Akka.Cluster"
  active-strategy = down-all
}
```

#### Keep Referee

This strategy is only available with the legacy Akka.Cluster split brain resolver, which you can enable via the following HOCON:

```hocon
akka.cluster {
  downing-provider-class = "Akka.Cluster.SplitBrainResolver, Akka.Cluster"
  active-strategy = keep-referee

  keep-referee {    
    # referee address on the form of "akka.tcp://system@hostname:port"
    address = ""
    down-all-if-less-than-nodes = 1
  }
}
```

> [!WARNING]
> Akka.NET's hand-rolled split brain resolvers are deprecated and will be removed from Akka.NET as part of the Akka.NET v1.5 update. Please see "[Split Brain Resolution Strategies](#split-brain-resolution-strategies)" for the current guidance as of Akka.NET v1.4.17.

The `keep-referee` strategy will simply down the part that does not contain the given referee node.

When to use it? If you have a single node which is running processes crucial to existence of the entire cluster.

Things to keep in mind:

1. With this strategy, cluster will never split into two independent ones, under any circumstances.
2. A referee node is a single point of failure for the cluster.

You can configure a minimum required amount of reachable nodes to maintain operability by using `down-all-if-less-than-nodes`. If a strategy will detect that the number of reachable nodes will go below that minimum it will down the entire partition even when referee node was reachable.

#### Lease Majority

The `lease-majority` downing provider strategy keeps all of the nodes in the cluster who are able to successfully acquire an [`Akka.Coordination.Lease`](xref:Akka.Coordination.Lease) - and the implementation of which must be specified by the user via configuration:

```hocon
akka.cluster.split-brain-resolver.lease-majority {
  lease-implementation = ""

  # This delay is used on the minority side before trying to acquire the lease,
  # as an best effort to try to keep the majority side.
  acquire-lease-delay-for-minority = 2s

  # If the 'role' is defined the majority/minority is based only on members with that 'role'.
  role = ""
}
```

A `Lease` is a type of distributed lock implementation.

> [!NOTE]
> We are working on improving the documentation for Akka.Coordination, which will shed some more light on how this feature can be used in the future.

### Relation to Cluster Singleton and Cluster Sharding

Cluster singleton actors and sharded entities of cluster sharding have their lifecycle managed automatically. This means that there can be only one instance of a target actor at the same time in the cluster, and when detected dead, it will be resurrected on another node. However it's important the the old instance of the actor must be stopped before new one will be spawned, especially when used together will Akka.Persistence module. Otherwise this may result in corruption of actor's persistent state and violate actor state consistency.

Since different nodes may apply their split brain decisions at different points in time, it may be good to configure a time margin necessary to make sure, that other nodes will get enough time to apply their strategies. This can be done using `akka.cluster.down-removal-margin` setting. The shorter it is, the faster reaction time of your cluster will be. However it will also increase the risk of having multiple singleton/sharded entity instances at the same time. It's recommended to set this value to be equal `stable-after` option described above.

#### Expected Failover Time for Shards and Singletons

If you're going to use a split brain resolver, you can see that the total failover latency is determined by several values. Defaults are:

* failure detection 5 seconds
* `akka.cluster.split-brain-resolver.stable-after` 20 seconds
* `akka.cluster.down-removal-margin` 20 seconds

This would result in total failover time of 45 seconds. While this value is good for the cluster of 100 nodes, you may decide to lower those values in case of a smaller one i.e. cluster of 20 nodes could work well with timeouts of 13s, which would reduce total failover time to 31 seconds.
