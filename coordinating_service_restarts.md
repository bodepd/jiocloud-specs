# Introduction

This specification outlines approaches that can be used to coordinate restart of
services across an application cluster.

# Problem Statement

Clustered application that need quorum to be in a healthy state need to
coordinate certain events that risk the cluster's ability to stay in
a healthy state.

The most obvious problem that we have is the following use case:

* When cluster adds a new member, configuration needs to be updated.
* For configuration to be applied, a restart must occur.
* If more than one node restarts at a time, there is risk of split brain,
  or other states when quorum cannot be achieved (and possibly even data
  corruption).

This issue can occur with ceph monitors, zookeeper, and possibly cassandra (am
I missing anything?)

# Proposed Solution

Use consul sessions to aquire locks on specific keys to ensure that only one
cluster member will restart at a time.

# Alternative Solutions (Pros/Cons)

We could build an external system to coordinate these types of events, but it
would violate our design goal of having all nodes be authoritative over the
actions the perform.

# Proposed implemenation

Add two methods to jorc:

* get\_lock - blocks until a lock can be aquired or times out
* release\_lock - release a lock if we are holding it

Create a script that wraps the service restart calls. This script should
accept the name of the service that it should restart (ie: zookeeper),
an optional command that should be called to determine that the lock
can be released (I don't think that we can trust service script completion
to indicate when it's safe to release a lock), and a timeout value.

The script should do the following

````
until timeout:
  get\_lock $name
  service $name restart
  until $done\_command:
    sleep $x
  release\_lock
````

# Alternative implementations

I considered how to design a system like this in Puppet:

* have a resource that blocks until a lock is aquired
* have a resource that releases that lock
* use deps to say that the lock should be acquired before the service is restarted which
  happens before the lock is released.

This solution has the following problems:

* if the service fails to restart, the lock will not be released
* relying on service restart to finish may not be robust enough to release lock
* the fact that sessions are baesd on uuids require Puppet to make an additional lookup

# Unknowns

For the case of zookeeper, I am not even sure if this resolve the consistency
issues that I am seeing.
