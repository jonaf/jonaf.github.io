---
title: Zone-aware load balancing for Ostrich
updated: 2016-12-28 22:45
---

I'm proud to announce that we've open-sourced a [library of load balancers][1] for [Ostrich][2], with our first load balancer, the zone-aware load balancer. This load balancer has an affinity for network-local service nodes, which improves performance by reducing network latency, and if you're using Amazon Web Services, may also reduce the cost of data transfer between availability zones. And it's fast: the load balancer must make many decisions very quickly, as it's used in a high-throughput system, so we measure its performance in nanoseconds (there's 1 million of those per *millisecond!*).

### History and motivation

At Bazaarvoice, we use [Ostrich][2] for mid-tier host discovery and load balancing. For several years (since its creation), the most common implementations of Ostrich have been very bare bones. Service providers expose a very small payload (mostly IP addresses and ports), and service consumers mostly take the complete set of endpoints from the service provider and distribute requests to the endpoints in uniformly-random fashion (emulating, in effect, round-robin load balancing, although the math is different).

Additionally, most of our infrastructure at Bazaarvoice is in Amazon Web Services (AWS), deployed across a minimum of three (3) availability zones. Availability zones are isolated data centers within one Amazon region. A typical setup for a service might look like this:

    [diagram showing 3 AZ's with 1 service node in each]

Not surprisingly, a service consumer will also be typically deployed in a similar fashion:

    [diagram showing 3 AZ's with 1 consumer node in each]

With uniformly random load balancing, each consumer node will choose each service node ~33% of the time. So a consumer node that is sending 1,000 requests per second would send 333 requests to the service node in its own zone, and 666 requests to the service nodes in other zones (333 each). This is great because you get even distribution of traffic to the service nodes, which makes scaling, monitoring and management of the service infrastructure quite simple -- none of the nodes or zones are treated specially. In fact, this is the ideal way to scale and manage systems. I won't go into all the details explaining why, but every system I've ever worked with (particularly distributed databases) work *really hard* at trying to evenly distribute ownership (Cassandra does a particularly great job at this with its Murmur3 partitioner).

There's a cost to this nice property of scaling our systems, however. That cost comes in two ways. The first way is the network latency. If my consumer node does 1,000 req/s, and 2/3 of those requests incur additional network latency because they are in other availability zones, and let's say that network latency is a mere 5 milliseconds, that means that I have spent a total of (666 * 5) = 3,330ms latency *per second.* You can see how wasteful this is. In a high-throughput system, this adds up very fast.

The second way we feel the cost of this crossing over to other zones is monetarily. Amazon charges $0.01 per GB for data transfer between any two availability zones (in the same region, of course). This may not seem very expensive, but again, in a high-throughput system, we routinely transfer in the range of terabytes across zones. It gets expensive, fast!

### Enter: zone-awareness

After some analysis, I found that we were spending an incredible amount of money and suffering a lot of network latency *completely unnecessarily.* And so began the effort to implement zone-aware load balancing.

### Ostrich crash course

I won't go into the details of how Ostrich works, as I'd be here writing all day, but the short version is that service nodes are Ostrich "service providers," which means that they register themselves using Ostrich into a discovery service. In our case, we use [Zookeeper][3]. So a service, let's call it "my_service," will start up and register itself as a service provider, creating an ephemeral Znode in Zookeeper with a path like:

    /ostrich/us-east-1/my_service/node_ip

We install a payload at this path, which tells us things like the IP of the node and the port we'll communicate on. We can also include other information (whatever we want), but this is all the information we need to discover and start making any RPC calls we want to a service.

So, the registration is where the work of the service provider stops. There's some other stuff we do, like providing health check endpoints and maintaining a Zookeeper connection so that the ephemeral node isn't automatically cleaned up. But that's mostly all the service provider has to do for consumers to begin discovering and calling it.

For a consumer, the job is also pretty simple. A consumer creates an Ostrich ServicePool and provides some information, like the service name, in order for Ostrich to perform discovery. This ServicePool is what supplies the endpoints for your RPC client (say, an HTTP client) to use when building requests or connections. When you build the ServicePool, you can supply a load balancing algorithm. If you don't supply one, the default one -- a uniformly random load balancer -- will be used.

### Enter: Zone-awareness (...again)

The zone-aware load balancer is effectively the algorithm supplied to a ServicePool builder. This way, when the ServicePool is constructed, all your RPC client does is ask for an endpoint, and the ServicePool gives you a (healthy) endpoint among the available ones. Though the default will just give you any random, valid endpoint, the zone-aware load balancer will look at all of the valid endpoints and their metadata, and return an endpoint that is (probably) in the availability zone of your client.

I said "probably." Why not always? Well, what happens when there are no valid endpoints in the same zone as the consumer? Right, when in doubt, the zone-aware load balancer will fallback to the random algorithm. But, that's not the only time we'll fallback, or even make some alternative decision. There are a lot of possible topologies we can run into. For example, what if the service nodes look like this?

    [service nodes, 3 AZ's, 2 in each zone but 1 in the third zone]

And if our consumer nodes looked like this:

    [consumer nodes, 3 AZ's, 2 in each zone]

Let's say that each service node has a maximum capacity of 100 req/s, and that each of our consumer nodes is sending traffic at 50 req/s. If we're doing random load balancing, the above topology would work out fine. Every node will receive ((3 * 2 * 50) / 5) = 60 req/s, regardless of its zone. However, if we're naively choosing the same zone whenever endpoints are available in it, then the first two zones will send ((2 * 50) / 2) = 50 req/s, but the consumer node in the last zone will send ((2 * 50) / 1) = 100 req/s, putting it at maximum capacity, but more importantly, this single service node is doing *twice* the work as its peers, which most likely negates any performance gains from network locality (and then some). Can you imagine diagnosing or mitigating performance problems in this scenario? How do you scale this system?

I mentioned earlier one of the nice properties of the random load balancer being ease of scale and management. This is one such example where a change to your load balancing algorithm can result in other problems.

    Side note: an astute reader may point out that the system may "work itself out," because the server at maximum capacity would fail health checks and be removed from the ServicePool. While this is true, it's also true that the node would periodically become healthy again, get re-added to the pool, and finally reach maximum capacity and become unhealthy again.

With our zone-aware load balancer, I needed to maintain this property of the random load balancer. To achieve this, we have to create a formula that takes into account the balance of service nodes in their availability zones, such that we can satsify both requirements: (a) affinity for network locality, and (b) perfectly even distribution of requests to service nodes. To add a little complexity to the mix, each consumer node's load balancer needs to be able to make the correct decision without knowing the zone or number of other consumer nodes -- we have to make this decision, quickly, only having information about the service nodes. Oh, and don't forget that we are never guaranteed to see, at all times, how many zones there are for service nodes -- a consumer will only know of the zones for service nodes seen so far, so it has to be adaptable to the addition (or removal) of zones.



[1]: https://github.com/bazaarvoice/ostrich-load-balancing
[2]: https://github.com/bazaarvoice/ostrich
[3]: https://zookeeper.apache.org/