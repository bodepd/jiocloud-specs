# intro

Intended to describe the requirements and design of asynchronous orchestration and
node lifecycle management solution.

The main design goal of this system is that it should be highly scalable and
the decision making process for each system should be as decentralized as possible.

This means that all logic related to what each individual machine should do
should reside on that individual machine. Centralized state will exist,
but it's main purpose is to provide data that each individual machine can
use to decide what action is should perform.

# definitions

There are several definitions that you must understand when building a solution
to this problem.

1. Data dependencies - In many cases, a machine cannot even configure itself
until it has data related to another machine that hosts a service that it relies
on.

2. Run time dependencies - Even once a machine gets the address of a service
   that is depends on, that address much map to a functional service in order
   for the role to become operational.

# orchestrating updates:

Our orchestration system needs to be able to support applying updates to the
cloud system. An update could be any change that needs to occur: a package upgrade
, a configuration update, or even a patch that we generated in house.

All of these things will be associated with a specific package repository snapshot.

## requirements

It needs to support the following:

- store the desired version of the package repository that should be used
- retrieve the current desired version of the package repository
- track which hosts are current running which version
- hosts need to be able to determine by themselves if there is an update and how to apply it

## Centralized data

The following centralized data should exist to track the state of upgrades.

### /current\_version

The desired version of the overall system.

````
current\_version => version
````

### /running\_version

Tracks the actual versions that are out in the wild.

````
/running\_version
- <version>
  - <node[1..n]>
````

## tools

### trigger\_update

Add a current\_version (this is consumed by all hosts to force an upgrade or
used for the installation process)

````
python -m jiocloud.orchestrate trigger\_update <version>
````

### verify\_hosts

Accepts a list of hosts and a version. Verifies that all specified hosts are
currently marked as running that version. Used to track the status of an update.

Ie: returns true if `/running\_version/<version>/<host>` exists in etcd for all
specified hosts.

````
python -m jiocloud.orchestrate --host <etcd\_address> verify\_hosts <version> <hosts>
````

### pending\_update

Determines if a host needs to apply an update or not (ie: figures out if his current
running version matches the desired version)

````
python -m jiocloud.orchestrate --discovery\_token=$discovery\_token pending\_update
````

## notes

what is the desired behavior if we trigger an updated version and not all machines are in the
desired state?

# build states

For the purposes of debugging this system and monitoring build progress, more
information is required than what version each system thinks it is running.

There are many states related to the status of Puppet, local service
validation, as well as overall cluster health.

## requirements

- keep track of what machines are not accounted for (what machines have
  been provisioned, but are not updating their status)
- get a list of machines whose configuration did not get correctly applied
- get a list of machines whose monitoring checks are not succeeding
- each machine should be able to update its own info
- for each status type (puppet, validation) machine can only be in one state at a time

## provisioning (we are not tackling this initially)

Machines should be marked as successfully provisioned once cloud-init
has finished processing and succeeded.

One a machine is in this state, it should only be removed once it has
been decommissioned which should be an explicit process.

The intention of this list is to keep track of what machines should exist
so this state can be reconciled against what systems are checking in in order
to locate unresponsive systems.

## configuration

The states related to applying configuration with Puppet
- configuration is currently being applied (do we care about this state?)
- configuration has been successfully applied that puts a machine into version X
- configuration has failed to apply on the last run
- configuration cannot be applied because required external services are not ready

## Validation

The actual observed state of a machine
- validation is currently being performed (do we care about this state?)
- a machine has failed to validate it is running the expected service
- a machine has been verified as running the expected service (this might just
be the same as updating its running\_version.)

## key architecture for maintaining node state information

Proposed key hierarchy for keeping track of machine states using etcd:

````
/status/
  - provisioned
    - success
       [hostlist]
    - failed
       [hostlist]
  - puppet
    - success
      - [hostlist]
    - failed
      - [hostlist]
    - running (might add this)
    - pending (not sure if this actually makes sense)
  - validation
    - success
      - [hostlist]
    - failed
      - [hostlist]
````

## tooling

### get\_failures

A tool that can determine:

1. are any hosts in a failed state
2. which hosts are in a failed state

### update\_status

A tool to update a hosts state. It should also ensure that host is only in
one state per category.

## notes

We will tackle the provisioned state stuff later.

Eventually, relevant logs should be sent to an object store when failures occur and
failures should link to these artifacts.

# service discovery and cross host dependencies

The entire point of configuring machines is so that they provide services.
Some services also need to consume other services to operate.

In order to implement flexible and scalable cross host orchestration of services,
machines need to be able to publish their running services, and other machines
need to be able to consume those published services.

Sometimes machines can't really be configured to run a service until another
machine has already been configured because the two hosts have a service dependency.

In general, these types of dependencies are all about connectivity to a running service.

This dependencies can be expressed as follows:

````
I cannot be configured until N of Service type X are addressable
````

````
I cannot be functional until N of Service type X are functional and reachable
````

This means that Additional information needs to be stored:

1. When services are in a successfully deployed state, they need to publish
   their connectivity information. (initially, we should assume this can be
   a single address)

2. Roles need to be able to express their external dependencies:
   - what services do we depend on
   - how many of them do we expect
   - which dependencies are on data available during discovery versus.
     running services

## the consul way

Service discovery and cross host dependencies are the main motivation for
moving things to consul. This section covers the basics of the consul solution.

### Service publication

Consul supports service publication via an integration with DNS.

#### define services

Services are defined using consul's json format:

````
{
  service => {
    name  => $name,
    port  => $port + 0,
    tags  => $tags,
    check => {
      script => $check_command,
      interval => $interval
    }
  }
}
````

Which is implemented as the following defined resource type in Puppet:

````
define rjil::jiocloud::consul::service
````

Each Puppet profile is responsible for registering its services that need
to be addressable within the OpenStack cluster. These services are registered
in DNS via A records of the form: `<service_name>.service.consul` and
`<service_name>.service.<data_center>.consul`.

For information on consul DNS can be found (here)[http://www.consul.io/docs/agent/dns.html].

#### Service Configuration Orchestration

##### Data dependencies

Since consul relies on DNS for addresses, configuring service to connect
to each other (ie: the data dependency) is a relatively easy problem.

In hiera, services can configure themselves by referring to the DNS name
of the registered service that they need to connect to.

Of coarse, those services being configured may need to use IP address for
configuration that needs to be performed before its services are registered
in DNS>

NOTE: For some use cases, the system needs to be able to query for
a list of things that are hosting a service. I am still learning more
about the stack to be configured to understand if this is a requirement.

##### Run time dependencies

Data dependencies are only 1/2 of the problem. Services don't only need
to be able to configure each other to interconnect, they often need to
be able to ensure that a remote service is running before they should
perform some actions.

For example, database schema migrations should not be run until there
is a functional database service. Also, keystone objects cannot be
populated until a keystone service is running and addressable.

There is a special resource in Puppet `jil::service_blocker` that is
used to specify the DNS name that it should wait for.

It trusts the fact that consul is managing DNS to allow bind to
functional endpoints, and assumes that any service that currently
has registered A records is an active service (as we get further into
the problem, we may find this has too many race conditions, and may
need to replace the simple DNS checks with actual service availability
checks)

The service blocker resource is intended to be added to your catalog.
It retries until the DNS address is either resolvable, or the resource
times-out. In order to work it needs to be set as a dependency for things
that depend on the service being available.

The following shows an example of the configuration of keystone blocking
until the mysql service is addressable.:

````
ensure_resource( 'rjil::service_blocker', 'mysql', {})
Rjil::Service_blocker['mysql'] -> Keystone_config['database/connection']
````

Note that it blocks the actual keystone\_config resource and not a class.
I have chosen that reason for the following reasons:

1. Choosing a high level class could more easily lead to deadlock issues
   (ie: circular dependencies)
2. I am trusting that things that depend on the database being functional
   are setting their own dependencies on this configuration correctly.

## tooling

### Write service discovery

When a machine thinks it is correctly running a certain version of a certain service, it
should publish the information somewhere that will be consumable by the rest of the cluster.

#### etcd way

The puppet defined resource type: rjil::profile { name: } writes $name
in a local file /var/lib/puppet/profile\_list.txt.

The following script:

````
python -m jiolcoud.orchestrate publish\_service [host] [addresses] [services...]
````
publishes all interfaces/addresses for each entry in that file.

The big question here is when to publish information related to the services.

Initially, I attempted to only publish information about services once those
services were correctly validated. This lead to a deadlock problem b/c of
circular cross host data dependencies.

For example: There is a circular cross host data dependency between a controller
and it's load balancer. The load balancer cannot correctly configure itself until
it knows the address of a pool member (ie: openstack controller), the controllers
cannot configure themselves until they know the address of the load balancer (b/c
each service should be interconnected through the load balancer to increase
reliability)

Eventually, I moved the publish code to happen after the initial Puppet run, regardless
of whether or not it actually succeeds. This ensures that all services will register
(as long as they do not have compile failures), it also ensures that all roles should
eventually converge to the desired cluster state.

### Consume service catalog

Each service that has an external dependency on a running service should be able to
consume the discoverable data about that service and configure itself based on
that data.

#### the etcd way

Before each run of Puppet, the command:

````
python -m jiocloud.orchestrate cache\_services
````

This command will convert the entire service catalog from etcd in the file

````
/etc/puppet/hiera/data/services.yaml
````

All entries are of the format:

````
#services::role::interface: [addr1, addr2]
services::glance::eth0: [10.0.0.5, 10.0.0.6]
````

These discovered services need to be mapped to the actual class parameters
using hiera, ie:

````
keystone\_public\_address: "%{lookup\_array\_first\_element('services::controller\_load\_balancer::eth0')}"
````

# Architectural issues

While the above stated approach based on eventually consistent configuration
based on availability of discovered services *will* work, it has several issues
that will make it difficult to build our solution on top of.

1. If the Puppet run does not compile, the service will not be configured and published.
This means that any compile failure related to data that should be available later
can result in a deadlocked state (where two nodes cannot compile Puppet and lay down
the configuration b/c they both depend on data from each other to compile). It is
possible to require defaults for all cross-host hiera data

2. In the eventually consistent model, the results are possibly non-deterministic.
This is because success/failure may depend in the order in which events happen
to occur between multiple hosts. (ie: the same code could work if host A happens
to publish it's service information before host B happens to run, and not work if
the hosts happen to run in the reverse order)

3. In the eventually consistent model, you cannot distinguish between a failed,
non-recoverable state and when a build just needs more time. The basic idea is
that Puppet fails and keeps rerunning until enough state exists about the network
for everything to work.

4. Currently, puppet will not run if the system believes that it is already
running the correct version. This has led to intermittent errors during testing
where machines enter an unrecoverable state if if they update their current version
before they are actually configured.

Ex1: a load balancer could be functional when it has one member, but it actually
needs to add an additional member.

Ex2: I sometimes see machines update their version, even if the service is not
functional. I don't understand how this happens.

## the flaw

After listing all of the known limitations that effect the usability of the above
mentioned system, they are all symptoms of a single design flaw within the system.

The system treats configuration as one giant step, as opposed to accounting for
the fact that we are applying three types of configuration.

1. configuration that should be consistent

Most of the configuration being applied is completely static. It should be
applied, and should always pass on the first try. Any other result should
be treated as a suspect system (which probably means that it should be
destroyed and re-provisioned from scratch)

2. configuration that is eventually consistent

Some of the configuration should eventually be correct based on the availability
of discoverable machines.

To connect this point to the previous, flaws, I will explain one-by-one how
they are a symptom of this issue:

1. Puppet compile failure deadlocks:

The issue here is that consistent data that has to exist (service discovery
profile data) is dependent on eventually consistent data.

2. Results are not deterministic

While this problem is hard to completely solve, it is exaggerate in this case
because there are many reachable states that may not be recoverable. In general,
these possible states exist because eventually consistent configuration is being
initially incorrectly applied while the overall system waits for published
services.

3. cannot distinguish between real failure and failure because of eventual
consistency.

Moving the consistent data to a separate process means that we can consider
it's failures to be much more meaningful than things related to eventual
consistency.

4. Dead end state because only configuring based on new updated version

We are not continuously running Puppet because it is too expensive. In fact,
the consistent parts of the configuration are what is expensive, the parts
related to service lookup are relatively cheap. Splitting these up, means
that we can just run the expensive Puppet parts when there is an update
and continuously run the remaining parts.

## solutions

Our service publication and discovery solution needs to be modified to ensure
that it provides a model that is easier to development against, test, and debug.

This section outlines some of the possible solutions.

### consul

[Consul](http://www.consul.io/) appears to be much closer to the solution
that we need than etcd. Consul implements both a key/value store as well as
a service discovery system that appears to support the features that had
previously been implemented using etcd as a backend.

The potential pitfall of consul, is that it works at a much higher level than
etcd (which is basically just a backend), therefor, we need to ensure that it
supports our use cases and provides us with teh reliability that we actually
need. It may take a little time before we have 100% confidence in consul.

The most viable solution with consul is to use it as a clear line between
consisten and eventually consistent configuration. In this model, Puppet's
responsibility is to model all of the things that can be guarenteed to be
configured without relying on external data or services. As a part of this
configuration, Puppet is also responsible for configuring consul, configuring
the validation checks used by consul to verify service avaibility, and
configuring services to be registered via consul.

In this model, it is consul's responsibility to configure everything related
to external services. In an ideal world, this simply means that consul can
use it's DNS service registration system to assign static DNS names to servies
as they are registered while Puppet uses those DNS addresses so that it can
statically configure everything.

####  the problem:

Not all configuration is as simple as registering and validating a service,
then updating DNS where the client can just use a single DNS address. Often,
the configuration entries require a list of ip addresses. In these cases,
something needs to respond to consul, then update configuration files
(and potentially restart services) in response to registered services in
consul.

NOTE: we can assign all hosts that provide a certain service (ie: ceph-mon)
will register a SRV record.

a single dns name, and ensure that they also have individual records with
a specific hostname. Then we could integrate hiera so that it can perform
a dns lookup to get ip addresses back. This will likely have to be a custom
backend.

In these cases, it does not make sense for the regular Puppet process to be
responsible for this configuration. This *might* mean that we would have to
supply large amounts of upstream patches to make the upstream modules compatible
with this approach. Otherwise, Puppet would revert configuraiton changes made
by consul.

NOTE: we could just accept this, make sure that everything is recoverable, and
have consul signal puppet to rerun when services change.

#### notes

I think there is a lot of value in having consul configuring services using Puppet.

1. Puppet already supports idempotency and state change tracking, useful information
to keep track of, even when services are configured via consul.

In this model, seperate classes are used to consume data that comes from consul,
and listeners respond to data updates in consul by generating and applying a
Puppet catalog based on that data.

### improve existing model

For the sake of time, it may make sense to have another iteration on the
model to make it more usable until we are ready for the complete consul
refactor. This section outlines what that might look like.

#### split out service registration

In order to avoid deadlock because of issues where we cannot register services
because of Puppet failures the service publication code could be split out from
the regular resources.

A way to do this is to create either a special Puppet class or a different site
manifest that can write the service publication code as part of a different
Puppet run. This ensures that services can be written even if they are awaiting
data from other services.

This could also enable us to have Puppet compilation fail until the correct data
is available.

#### specify depdencies and failures in Puppet

#### slowly migrate to consul

Even if we decided to leave the service publication code for now, we could
easily start migrating the easier use cases to consul.

1. For things that have a single DNS name, use consule to register that service
into DNS. This will simplify the Puppet code that is looking up addresses from
the discoverable services.

2. Start to individually port services that need to be configured with multiple
addresses to consul.

# other notes

if validation is false, always rerun puppet

write a DNS hiera lookup so that we can resolve the same hostname into multiple entries

register a forward and reverse DNS lookup, so that we can say:

  get all cephmon -> get addresses -> per address, lookup unique dnsname
