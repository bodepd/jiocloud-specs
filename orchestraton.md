# intro

Intended to describe the requirements and design of key based orchestration and
node lifecycle status tracking implemented in etcd.

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

## etcd implementation

### keys

#### /current\_version

Intended to be the desired version of the system.
This assumes that the desired state of the entire infrastructure controlled
by an etcd cluster

````
current\_version => version
````

#### /running\_version

Tracks the actual versions that are out in the wild.

````
/running\_version
- <version>
  - <node[1..n]>
````

### tools

#### trigger\_update

Add a current\_version (this is consumed by all hosts to force an upgrade or
used for the installation process)

````
python -m jiocloud.orchestrate trigger\_update
````

#### verify\_hosts

Accepts a list of hosts and a version. Verifies that all specified hosts are
currently marked as running that version.

Ie: ruturns true if `/running\_version/<version>/<hostn>` exists in etcd for all
specified hosts.

````
python -m jiocloud.orchestrate --host <etcd\_address> verify\_hosts <version> <hosts>
````

#### pending\_update

Determines if a host needs to apply an update or not (ie: figures out if his current
running version matches the desired version)

````
python -m jiocloud.orchestrate --discovery\_token=$discovery\_token pending\_update
````

### notes

what is the desired behavior if we trigger an updated version and not all machines are in the
desired state?

# failure states

For the purposes of debugging this system, we need to know much more than
just what version each system thinks it is running. We also need to track
which systems have gone into unknown states.

## requirements

- keep track of what machines are not accounted for (what machines have
  been provisioned, but are not updating their status)
- get a list of machines whose configuration did not get correctly applied
- get a list of machines whose monitoring checks are not succeeding
- each machine should be able to update its own info
- for each status type (puppet, validation) machine can only be in one state at a time

### provisioning (we are not tackling this initially)

Machines should be marked as successfully provisioned once cloud-init
has finished processing and succeeded.

One a machine is in this state, it should only be removed once it has
been decommissioned.

The intention of this list is to keep track of what machines should exist
so this state can be reconciled against what systems are checking in in order
to locate unresponsive systems.

### configuration

The states related to applying configuration with Puppet
- configuration is currently being applied (do we care about this state?)
- configuration has been successfully applied that puts a machine into version X
- configuration has failed to apply on the last run
- configuration cannot be applied b/c required external services are not ready

### Validation

The actual observed state of a machine
- validation is currently being performed (do we care about this state?)
- a machine has failed to validate it is running the expected service
- a machine has been verified as running the expected service (this might just
be the same as updating its running\_version.)

## lifecycle using etcd

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

Eventually, relevent logs should be sent to an object store when failures occurr and
failures should link to these artifacts.

# service discovery and cross host dependencies

The entire point of configuring machines is so that they provide services.
Certain services also need to consume certain services to operate.

In order to implement flexible and scalable cross host orchestration of services,
machines need to be able to publish their running services, and other machines
need to be able to consume those published services.

Sometimes machines can't really be configured to run a service until another
machine has already been configured b/c the two hosts have a service dependency.

In general, these types of dependencies are all about connectivity to a running service.

This dependencies can be expressed as follows:

````
I cannot be configured until N of Service type X are addressable
````

This means that two types of information need to be stored:

1. When services are in a successfully deployed state, they need to publish
   their connectivity information. (initially, we should assume this can be
   a single address)

2. Roles need to be able to express their external dependencies:
   - what services do we depend on
   - how many of them do we need

## tooling

### Write service discovery

When a machine thinks it is correctly running a certain version of a certain service, it
should publish the information somewhere that will be consuable by the rest of the cluster.

````
python -m jiolcoud.orchestrate publish\_service [host] [addresses] [services...]
````

### Consume service catalog

Consume the list of services along with their ips from etcd. Write this data
to hiera in a well-defined format.


## proposed etcd solution

Ideally, all cross host orchestration can be simplified just to the point where each service needs
to be able to query for the ip addresses of hosts that fullfill a desired role.

This assumes that there are no circular dependencies (cases where two services need data from
each other)

This information can be written in the following key structure:

````
/active\_roles/role/host => address
````

This information should be pulled into hiera. Perhaps just as a well known namespace. It would likely
be pulled in from a hiera etcd backend.

````
services::<role>::<interface>: [addr1, addr2, addr3]
````

Roles should search their external data dependency constraints:

````
node /somerole/ {
  if hiera('etcdlookup::active\_roles::db').size < 2 {
    custom_fail_method
  } else {
    # configure stuff with Puppet
  }
}
````

# notes
