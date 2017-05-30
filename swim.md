# SWIM (Scalable Weakly-Consistent Infection-Style Process Group Membership) Protocol

#### https://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf

## Previous work

- Previous implementations rely either on:
  - Sending periodic heartbeats to every other node in the cluster
    - quadratic as nodes are added to cluster
  - Sending periodic heartbeats to a subset of nodes
    - Requires maintaining an overlay graph
