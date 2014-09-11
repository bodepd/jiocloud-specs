# intro

Intended to describe the requirements and design of key based orchestration and
node lifecycle status tracking implemented in etcd.

# orchestrating updates:

In order to orchestreate updates, the key structure is pretty easy:

## keys

### /current\_version

Intended to be the desired version of the system.
This assumes that the desired state of the entire infrastructure controlled
by an etcd cluster

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
python -m jiocloud.orchestrate trigger_update
````

### verify\_hosts

Accepts a list of hosts and a version. Verifies that all specified hosts are
currently marked as running that version.

Ie: ruturns true if `/running\_version/<version>/<hostn>` exists in etcd for all
specified hosts.

````
python -m jiocloud.orchestrate --host <etcd\_address> verify\_hosts <version> <hosts>
````

## notes

what is the desired behavior if we trigger an updated version and not all machines are in the
desired state?

# manage machine lifecycle

Keeping track of the current version as well as the current running versions of all machines
is not enough information to actually understand the current state of an environment.

## machine lifecycle

The following list is intended to capture all of the possible states that a machine might
be in during its lifecycle.

### provisioning
States related to the initial provisioning of a machine
- successfully provisioned
- currently running cloud-init
- successfully completed cloud-init
- cloud init failed

### configuration
The states related to applying configuration with Puppet
- configuration is currently being applied (do we care about this state?)
- configuration has been successfully applied that puts a machine into version X
- configuration has failed to apply on the last run

### Validation
The actual observed state of a machine
- validation is currently being performed (do we care about this state?)
- a machine has failed to validate it is running the expected service
- a machine has been verified as running the expected service (this might just
be the same as updating its running\_version.

## lifecycle using etcd

Keep track of machine states using etcd

Here is a proposed key hierarchy:

````
/status/
  - provisioned
    - success
       [hostlist]
    - failed
       [hostlist]
  - configured
    - success
      - version1
        - [hostlist]
      - version2
        - [hostlist]
    - failure
      - version1
        - [hostlist]
  - validated
    - success
      - version
        - [hostlist]
    - failure
      - version
        - [hostlist]
````

### provisioned

Each host in the success state for provisioned should have an entry with no ttl.
These can be cleaned up manually when hosts are removed.

### service state

Each host should only even be in one of the following states:
  configured/success
  configiured/failure
  validated/failure
  validated/success

Whatever service updates hosts should remove its previous entry.

Each leaf may itself be a key that points to a link on swift to more verbose logs related to the event

## notes

we are initially not going to worry about the provisioned state where we can't just assume that cloud-init
will work. We can tackle this later.

# gathering cross node information

Ideally, all cross host orchestration can be simplified just to the point where each service needs
to be able to query for the ip addresses of hosts that fullfill a desired role.

This information can be written in the following key structure:

````
/active\_roles/version/<host>/address
````

This information should be pulled into hiera. Perhaps just as a well known namespace. It would likely
be pulled in from a hiera etcd backend.

````
etcdlookup::active\_roles::db:
  host1:
    eth0: foo
    eth1: bar
````

This namespace may not work perfectly b/c you may need to perform recursive hiera lookups to resolve data
from it and you cannot suppor things like:

````
glance::api::db\_ip: "%{hiera('etcdlookup::active\_roles').collect{|x| x['eth2']}}"
````

I guess we could patch hiera to support things like that. Otherwise the data mappings would have
to exist in Puppet which breaks the class param -> hiera data model.

Roles need to be able to know how to parse this hiera data to make a decision about what to do:

````
node /somerole/ {
  if hiera('etcdlookup::active\_roles::db').size < 2 {
    custom_fail_method
  } else {
    # configure stuff with Puppet
  }
}
````

The reason for failing via a custom method is b/c it needs to be parsed by the calling script to
make a determination if it should be markes in the pending state (waiting for external deps)

# notes
