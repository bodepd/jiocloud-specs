# introduction

Our monitoring requirements are driven by two factors:

1. scale

2. repeatability of monitoring processes as a part of the
   developmemt process

3. operations and development requirements - we need to ensure
   that we meet all requirements of operations and development

# Asumptions

1. A traditional monitoring tool cannot scale b/c it
   tries to maintain too much centralized state (or perform
   too many centralized tasks).

2. Centralized monitoring makes it too hard to have monitoring
   serve as way to test code as you develop.

3. As much logic as possible should be performed on the individual
   nodes so that the load of testing is distributed as much as possible.

4. Metrics should be backed-up in a place specific to each node

# Proposed Solution

## types of monitoring checks:

Seperate checks into the appropriate categories:

* service checks - used as service checks in consul. They are responsible
  for determinig if an individual service is working. They should not have
  any external dependencies on any other services.

* validation checks - responsible for determinig rather or not the core
  services are available. They can rely on services running on other
  machines. During deployments, they are used to satisfy eventual consistency
  requirements and tell nodes that they should continue running b/c things
  are not in the overall desired state.

## metric colletion

  metrics - metrics are used to capture time series information that can be
  used to determine how a service is performing over time. It should be combined
  with analysis tools run on each node that can identify when metric trends
  indicate an issue.

QUESTION: Am I missing any kinds of checks?

# Implementation

## validation

Deploy all monitoring checks using Puppet as bash scripts that live
in either the `/usr/lib/jiocloud/tests` in either the validation or
service\_checks directories dependening on the kind of check.

These scripts should comform the nagios standard return codes
(0=pass,1=warn,2=fail) as this is also what consul expects.

Register all services that must be alive in consul. Register a separate service
called validation that is used to register status of validation in consul.

Use the values of these checks to determine if we think a specific node is
healthy. Run Puppet on nodes that are not currently in a healthy state.

## Helper code

- get\_failures - used to determine if any nodes have any services in a
  failed state. This is used by jenkins to determine if a build passed.
- local\_health - used to determine if anything is failing on the current node.
  This is used to determine if a node should re-run Puppet.

## metrics

I am currently considering collectd for metric collection. It seems to be
the most widely used solution and has a very robust set of existing plugins.

We can additionally consider statsd if we find that we need to add a lot of
custom metrics.

Initially, we can just start collecting metrics and storing them locally while
we figure out more complicated things like how to support queries for
correlation and alerts without a centralized service.

Since metrics may need to persist when a node days, we can ensure that each
host has it's own object store where it can periodically backup it's metrics.

It would also be easy enough to host a local graphite instance of each node
so that you can check graphs of metrics per node (as a starting point)

I'm not sure about how alerting for metrics should work. Maybe we could register
a metrics service in consul? Otherwise, it might be nice to start by pushing
metrics to an external service until we figure out what our longer term solution
is.

## Alerts

I have not fully thought out how alerts will be implemented, here is my best
idea at the moment (which I could easily be convinced to change)

Consul already has services for indivual running services as well as
validation. We could easily add another

## Metric quiers/correlation

Also, not sure how this would work. We need to get more requirements. I would
like it to be a facility where you can query questions across the fleet and
have each node search it's local data to understand what to do. I did a bit
of research for tools that work like this and didn't immediately find
something.

# Considerations

- What nagios plugins can we reuse?
- What collectd plugins can we reuse?
- We should just reuse the enovance nagios plugins:
    https://github.com/enovance/openstack-monitoring

# Unknowns
