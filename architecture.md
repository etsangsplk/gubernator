## Gubernator Architecture

![architecture diagram](/images/architecture.png)

Gubernator is designed to run as a distributed cluster of peers which utilize
an in memory cache of all the currently active rate limits, as such no data is
ever synced to disk. Since most network based rate limit durations are held for
only a few seconds losing the in memory cache during a reboot or scheduled
downtime isn't a huge deal. For Gubernator we choose performance over accuracy
as it's acceptable for a small subset of traffic to over request for a short
period of time (usually seconds) in the case of cache loss.

When a rate limit request is made to Gubernator the request is keyed and a
consistent hashing algorithm is applied to determine which of the peers will be
the owner of the rate limit request. Choosing a single owner for a rate limit
makes atomic increments of counts very fast and avoids the complexity and
latency involved in distributing counts consistently across a cluster of peers.

Although simple and performant this design could be susceptible to a thundering
herd of requests since a single coordinator is responsible for possibly
hundreds of thousands of requests to a rate limit. To combat this, clients can
request `Behaviour=BATCHING` which allows peers to take multiple requests within
a specified window (default is 500 microseconds) and batch the requests into a
single peer request, thus reducing the total number of over the wire requests
to a single Gubernator peer tremendously.

To ensure each peer in the cluster accurately calculates the correct hash for a
rate limit key, the list of peers in the cluster must be distributed to each
peer in the cluster in a timely and consistent manner. Currently Gubernator
supports using etcd or the kubernetes endpoints API to discover gubernator
peers.

## Gubernator Operation
When a client or service makes a request to Gubernator, the rate limit config
is provided with each request by the client. The rate limit configuration is
then stored with the current rate limit status in the local cache of the rate
limit owner. Rate limits and their configuration that are stored in the local
cache will only exist for the specified duration of the rate limit
configuration. After the duration time has expired, and if the rate limit was
not requested again within the duration it is dropped from the cache.
Subsequent requests for the same `name` and `unique_key` pair will recreate the
config and rate limit in the cache and the cycle will repeat. Subsequent
requests with different configs will overwrite the previous config and will
apply the new config immediately.

## Global Behavior
Since Gubernator rate limits are hashed and handled by a single peer in the
cluster. Rate limits that apply to every request in a data center would result
in the rate limit request being handled by a single peer for the entirety of
the data center. For example, consider a rate limit with
`name=requests_per_datacenter` and a `unique_id=us-east-1`. Now imagine that a
request is made to Gubernator with this rate limit for every http request that
enters the `us-east-1` data center. This could be hundreds of thousands,
potentially millions of requests per second that are all hashed and handled by
a single peer in the cluster. Because of this potential scaling issue
Gubernator introduces a configurable behavior called `GLOBAL`.

When a rate limit is configured with `behavior=GLOBAL`, the rate limit request
that is received from a client will not be forwarded to the owning peer but
will be answered from an internal cache handled by the peer who received the
request. Hits toward the rate limit will be batched by the receiving peer and
sent asynchronously to the owning peer where the hits will be totaled and
`OVER_LIMIT` calculated. It is then the responsibility of the owning peer to
update each peer in the cluster with the current status of the rate limit, such
that peer internal caches routinely get updated with the most current rate
limit status from the owner.

#### Side effects of global behavior
Since Hits are batched and forwarded to the owning peer asynchronously, the
immediate response to the client will not include the most accurate remaining
counts. As that count will only get updated after the async call to the owner
peer is complete and the owning peer has had time to update all the peers in
the cluster. As a result the use of GLOBAL allows for greater scale but at the
cost of consistency. Using `GLOBAL` can increase the amount of traffic per rate
limit request if the cluster is large enough. GLOBAL should only be used for
extremely high volume rate limits that don't scale well with the traditional
non `GLOBAL` behavior.



