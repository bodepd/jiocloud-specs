# Introduction

This specification outlines a service that can be used to ensure that the
correct number of machines exists at all times. Currently, this is managed
by the Jenkins build process which runs about every 30 minutes.

In order to be more reactive, we need a separate service that can respond
to changes as they occur to make decisions about what changes need to
occur in our infrastrcuture.

# Problem Statement

Our environment needs to be able to maintain the correct number of each service
at all times to guarentee the following:

* services can handle the current amount of load (autoscaling)
* services are re-provisioned as soon as instances die
* if we move towards a model where we periodically and randomly kill instances
(ie: the [chaos monkey](http://techblog.netflix.com/2012/07/chaos-monkey-released-into-wild.html) style)

# Proposed Solution

Expand out current apply resources program to run as a background service on
one of the existing roles. It should be installed as a part of the build
and then activated as a part of the build process as soon as all services
have reached the validated state (b/c it is not something that we want to
run as a part of our intial bootstrapping).

This service would be responsible for the following:
* monitor for nodes that have stopped providing a service
* monitor for nodes whose metrics indicate some kind of an issue
* automatically specify the number of each type of node that needs to
  exist based on overall system health (like autoscaling)
* continuously ensure that the correct number of each instance is running

The service may be responsible for the following:
* making decisions about when to kill nodes
* * perhaps it could monitor nodes to see which ones might not be health and
    replace and kill them
* * I would also like it to provide a service like chaos monkey, but perhaps
    based on specifying a TTL for nodes (like all nodes can only exist for 1
    week)

# Alternative Solutions (Pros/Cons)

The obvious alternative is to install and use HEAT.

For background, we built apply\_resources and did not use heat for a few
reasons:
* the biggest reason is b/c heat requires installing an API service and other
services which we had not done yet. Writing a service that did the same was
easier than figuring out how to install/manage HEAT.
* I found heat to be a bit klunky when I tried to use it.

Going forward, it's definitley worth looking at HEAT as our solution for this.
We can start making this evaluation as soon as we install HEAT as a part of
both our under and overcloud.

I have a feeling that we will never migrate from something custom to heat, for
the following reasons:

1. I have a feeling that we can write the entire application that we need in about
500-1000 lines of code, heat is currently 140000 lines of code (this maybe a bit naive
of an estimate and include vendored soruce code, I ran:
`find heat/* | grep py | xargs cat  | wc -l` to get this data).
2. I have tried to patch HEAT to add simple features before and found the code
to be uncomprehensable. This maybe due to by novice level of python, but at the
same time, I have no problem adding the features we need to apply\_resources.
Given our timelines for needing this, training operations people to patch heat
to get it to do what we want seems unreasonable.
3. We need our application to integrate with consul (which is support we would
have to add to heat)

On the other hand, heat may be easier to provide as a service to our customers
in which case it may be worth the pain of either using it or modifying it to
be something that we can use.

Regardless of if heat is the right solution or not, it is unlikely that it will
be the initial implementation due to how soon we need this feature.

# Proposed implemenation

NOTE: I have no idea what the implemenation should look like, but I wanted
to capture enough requirements as possible.

Use the current apply\_resource approach for bootstrapping all initial nodes
as a part of the build environment.

Install a node lifecycle management service (maybe in the bootsrap node?)

This service requires credentials, but it would be nice it it was running a
consul agent. I would like it to run internally as a part of the build, but
could understand if we don't want credentials to live in the build.

Code this service to perform all desired actions.

Register metric heatlh in consul as a service (is this abusing cosul?)

# Deviations from Design guidelines

This will definitley require an additional centralized service

# Unknowns

It all seems pretty straight forward from an implementation perspective.
