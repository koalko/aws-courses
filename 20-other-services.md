# Other Services & Features

## AWS Local Zones

Availability Zones might be 100s of KM from business premises. Even with fibre connections, latency and performance impacts are noticeable.

With Local Zones, there is a Parent Region (with some AWS resources). For example, if Parent Region is `us-west-2` with `us-west-2a`, `us-west-2b` and `us-west-2c` AZs, Local Zones might be called `us-west-2-las-1` (Las Vegas), `us-west-2-lax-1a`, `us-west-2-lax-1b` (Los Angeles). Local Zones have their own connection to the internet, and also support `DX` (DirectConnect).

VPC can be extended to include subnets in local zones. Resources in local zones benefit from super low latencies (minimal distance to premises).

Local zone's EBS snapshots uses S3 in parent region (and replicated across all AZs within a parent region).

Local zone is 1 additional zone; so, no built-in resilience. Think of them like an AZ, but near your location.

Not all AWS products support local zones; many are opt-in with limitations.

DX to a local zone is supported (for extreme performance needs).

Local zone utilizes the parent region for some things (for example, for EBS snapshots).

Use local zones when you need the highest performance.