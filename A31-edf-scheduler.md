EDF scheduler for per endpoint weight
----
* Author(s): Yi-Shu Tai
* Approver: a11r
* Status: Draft
* Implemented in: N/A
* Last updated: 2020-08-05
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

The proposal introduces weighted round robin policy based on earliest deadline first scheduling algorithm for per lb_endpoint weight from eDS response.

## Background

gRPC provided xDS support in A27. It allows users to build their own control plane and route traffic more efficiently. In current state of implementation, xDS client takes per zone weight into account and load balancing by weighted_target policy which implements a random based weighted-roundrobin algorithm.


### Related Proposals:
* A27

## Proposal

The proposal is to take lb_endpoint weight into account and implement a weighted round-robin policy based on Earliest deadline first scheduling algorithm picker (EDF) (https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling).
examples,
```
eDS response with lb_endpoints: [("a", 2), ("b", 4)]
Picker returns: "babbabbabbabba"

eDS response with lb_endpoints: [("a", 2), ("b", 2)]
Picker returns:
"abababababababababababababab"
```

Summary of algorithm:

EDF scheduler maintains a priority queue of EdfEntry. Each entry has a pair (deadline,  order offset). On the top of the queue, it’s the entry with lowest deadline. order_offset is the tie breaker when two entries have same deadline, we want to pick the one which last picked is further.


- On each call to the Pick, EDF picker picks the entry e on the top of the queue, returns the subchannel associated with the entry. After that, picker update the (deadline, order offset) to (e.deadline + 1/weight, e.order_offset+1) and either perform a pop and push the entry back to the queue or key increase operation.
- If all endpoints have the same weight, EDF picker degenerates to RoundRobin picker
- Queue is rebuilt on every change to subchannel connectivity and weight, so the weight is effective in real time.

```
class EdfEntry {
    double deadline_;
    int order_offset_;
    RefCountedPtr<SubchannelInterface> entry_;
}

bool operator<(const EdfEntry& other) const {
    return deadline_ > other.deadline_ ||
        (deadline_ == other.deadline_ && order_offset_ > other.deadline_);
}
```

## Rationale

This feature enable users to take advantage of real time weight change from control plane to shift traffic accordingly to endpoints with lower load or away from bad endpoints.
The reason to introduce a new algorithm instead of using existing weighted target policy is that when all endpoint weights are the same, the picker should degenerate to round robin picker. We can argue that random-based weighted roundrobin is fine for per zone load balance because of high traffic.

## Implementation

Core, Go

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not  know the solution. This section may be omitted if there are none.]
