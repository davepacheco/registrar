<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2017, Joyent, Inc.
-->

# Registrar

This repository is part of the Joyent Manta and Triton projects. For
contribution guidelines, issues, and general documentation, visit the main
[Triton](http://github.com/joyent/triton) and
[Manta](http://github.com/joyent/manta) project pages.

Table of Contents:

* [Service discovery in Triton and Manta](#service-discovery-in-triton-and-manta)
* [Health checking](#health-checking)
* [Operating Registrar](#operating-registrar)
* [Developing with Registrar](#developing-with-registrar) (includes
  configuration reference)
* [ZooKeeper data format](#zookeeper-data-format)


## Service discovery in Triton and Manta

Triton and Manta components generally discover their dependencies through DNS.
There are three main components that make up this service discovery system:

- a ZooKeeper cluster, which keeps track of the list of instances of each
  different type of component
- Registrar (this component), a small agent that runs alongside most Triton and
  Manta components to register that component's presence in ZooKeeper
- [binder](https://github.com/joyent/binder), a server that answers DNS queries
  using the ZooKeeper state as the backing store

Let's examine what happens when an operator deploys a new instance of the Manta
service called "authcache".  We'll assume the deployment uses DNS suffix
"emy-10.joyent.us":

1. The operator provisions a new instance of "authcache".  This creates a new
   SmartOS zone (container).  The new container gets a uuid.  In this example,
   the uuid is `a2674d3b-a9c4-46bc-a835-b6ce21d522c2`.
2. When the new zone boots up, various operating system services start up,
   including the authentication cache itself and Registrar (this component).
3. Registrar reads its configuration file, which specifies that it should
   register itself at "authcache.emy-10.joyent.us".
4. Registrar connects to the ZooKeeper cluster and inserts a ZooKeeper ephemeral
   node called
   `/us/joyent/emy-10/authcache/a2674d3b-a9c4-46bc-a835-b6ce21d522c2`.  The
   contents of this node are a JSON payload that includes the IP address of this
   zone, as well as the ports it's listening on.
5. Some time later, a client of authcache does a DNS query for
   `authcache.emy-10.joyent.us` using its configured nameservers, which are
   running instances of `binder`.  Assuming binder doesn't have the answer to
   this query cached, it fetches the state out of ZooKeeper under
   `/us/joyent/emy-10/authcache`.  From this, it finds the IP addresses and
   ports of the authcache instances, including those of our newly-provisioned
   zone `a2674d3b-a9c4-46bc-a835-b6ce21d522c2`.  Binder caches this information
   for subsequent queries.
6. Binder translates this information into the appropriate DNS answers (usually
   `A` records or `SRV` records, depending on what the DNS client asked for).

If the zone is unprovisioned, or the server on which it's running reboots or
powers off or becomes partitioned, Registrar will become disconnected from
ZooKeeper.  This causes its ephemeral node to disappear from ZooKeeper.  Once
the Binder caches expire and they re-fetch the state from ZooKeeper, they will
no longer find information about the zone that's gone, so they will stop
including that zone in DNS answers.  Clients will shortly stop using the zone.

In this way:

- clients of a service (like "authcache") discover instances using DNS
- instances are added to DNS automatically when they start up
- instances are removed from DNS automatically for many (but not all) failures

Note that even in the best of cases, there is a non-trivial delay between when
an instance fails and when clients see that and stop using it.  For the instance
to fall out of DNS, Registrar's ZooKeeper session timeout must expire (usually
30-40 seconds), Binder's cache must expire (currently another 60 seconds), and
the DNS records fetched by clients must also expire (usually another 30-60
seconds, but this is configurable per-service, and it may actually take much
longer than that).  **Clients must still deal with failures of instances that
are present in DNS.**  DNS is the way that clients discover what instances they
might be able to use, but it should not be assumed that all those instances are
currently operating.

Of particular note, Registrar runs separately from the program that actually
provides service (the actual authentication cache process, in this example).  So
it's also possible for the real service to crash or go offline while registrar
is still online.  Health checking can in principle be used to mitigate this, but
it's currently buggy.

Many services (like [Moray](https://github.com/joyent/moray) use multiple
processes in order to make use of more than one CPU.  The recommended way to
implement this is to configure registrar to publish specific port information.
This allows binder to answer queries with SRV records, which allow clients to
discover not just individual zones, but the specific ports available in those
zones.


## Health checking

Registrar supports basic health checking by executing a command periodically and
unregistering an instance if the command fails too many times in too short a
period.  However, as of this writing, this mechanism is extremely buggy.  See
[HEAD-2282](http://smartos.org/bugview/HEAD-2282) and
[HEAD-2283](http://smartos.org/bugview/HEAD-2283).


## Operating Registrar

### Configuration

The configuration file is almost always immutable and based on a template that's
checked into the repository of a component that uses registrar.  Details are
described under "Developing with Registrar" below.

### Removing instances from service

There are many reasons why it's useful to remove an instance from DNS so that
clients stop using it:

- in development, this is useful to direct traffic at particular instances, or
  to test client behavior when instances come and go
- in production, this is useful to isolate malfunctioning instances.  You can
  remove these instances from service without actually halting them so that you
  can continue debugging them.

The usual way to do this is to disable the registrar SMF service in the zone you
want to remove from DNS:

    svcadm disable -st registrar

You can bring this zone back into DNS using:

    svcadm enable -s registrar


## Developing with Registrar

### Incorporating Registrar into a component

As mentioned above, Registrar is deployed in nearly all Triton and Manta
component zones.  Incorporating Registrar into a new component usually involves
only a few steps:

- the component's repository should include a template configuration file,
  usually in `sapi_manifests/registrar/template` and an associated config-agent
  snippet in the same directory
- the [Mountain Gorilla](https://github.com/joyent/mountain-gorilla) (build
  system) configuration snippet for this component should depend on the
  registrar tarball

In the most common case, the Registrar configuration file is itself generated by
[config-agent](https://github.com/joyent/sdc-config-agent) using the template
that's checked into the repository.  In some cases (notably Moray), there's an
additional step either at build time or zone setup time where the template
itself is templatized by the list of TCP ports that should be exposed.

When new instances (zones) of the component start up, config-agent writes the
registrar configuration file, populating variables such as the DNS domain suffix
(`emy-10.joyent.us` in the example above) from the SAPI configuration for the
current deployment.  After that, Registrar starts up, reads the configuration
file, and runs through the process described at the top of this README.


### Configuration reference

The configuration file is specified using the `-f` argument to "main.js".  The
file is a JSON object with top-level properties:

Property         | Type            | Description
---------------- | --------------- | -----------
`"adminIp"`      | optional string | IPv4 address to include in DNS records
`"registration"` | object          | describes which DNS names should be created
for this component
`"zookeeper"`    | object          | describes how to connect to the ZooKeeper cluster

**Address:**  The address for all DNS answers related to this component is
whatever is provided by the `"adminIp"` configuration property.  This is the IP
address used by clients that use DNS to discover this component.  For Triton
components, this should usually be an addresss on the "admin" network.  For
Manta components, this should usually be an address on the "manta" network.
If `"adminIp"` is not specified in the configuration, then Registrar picks an
up, non-loopback IP address on the system and uses that, but this is not
recommended.

**ZooKeeper configuration:** Service discovery records are maintained in a
ZooKeeper cluster.  The `"zookeeper"` top-level property describes how to reach
that cluster.  This should be a configuration block appropriate for
[node-zkplus](http://github.com/mcavage/node-zkplus).  See that project for
details, but essentially this needs to have a `"timeout"` and `"servers"`.
`"servers"` should be an array of objects with `"host"` and `"port"` properties
identifying the servers in the ZooKeeper cluster.

**Registration:** The `"registration"` object describes the service discovery
records that will be inserted into ZooKeeper.  These control the DNS names that
are available for this component.  Broadly, there are broadly types of service
discovery records:

* **Host records** essentially allow Binder to answer DNS "A" and "SRV" queries
  with the IP address (and possibly port numbers) for a single instance.  More
  precisely, host records are individual nodes in the ZooKeeper namespace that
  provide address and port information for a single zone.  These are ephemeral,
  which means they disappear when Registrar's ZooKeeper session expires.  That's
  by design so that if a zone disappears, it stops showing up in DNS.
* **Service records** allow Binder to answer DNS "A" and "SRV" queries for a
  single logical service that's provided by any number of interchangeable
  instances.  The list of instances available are represented by host records
  that are child nodes of the service record (within the ZooKeeper namespace).

Let's look at an example.  In Manta, instances of the "authcache" service
publish host records under $zonename.authcache.$suffix.  As a result, if you
have a Manta deployment whose DNS suffix is "emy-10.joyent.us", you can find the
IP address for the "authcache" zone that's called
`a2674d3b-a9c4-46bc-a835-b6ce21d522c2` by looking up
`a2674d3b-a9c4-46bc-a835-b6ce21d522c2.authcache.emy-10.joyent.us`:

    $ dig +nocmd +nocomments +noquestion +nostats a2674d3b-a9c4-46bc-a835-b6ce21d522c2.authcache.emy-10.joyent.us
    a2674d3b-a9c4-46bc-a835-b6ce21d522c2.authcache.emy-10.joyent.us. 30 IN A 172.27.10.62

We've just looked up the _host record_ for a particular authcache instance.  The
authcache service also writes a _service record_ for the higher-level DNS name
`authcache.emy-10.joyent.us`.  This lets clients of the authcache service use
the DNS name `authcache.emy-10.joyent.us` to find all available authcache
instances:

    $ dig +nocmd +nocomments +noquestion +nostats authcache.emy-10.joyent.us 
    authcache.emy-10.joyent.us. 30  IN      A       172.27.10.67
    authcache.emy-10.joyent.us. 30  IN      A       172.27.10.62

**Summary:** A service can provide host records (when there's only one IP
address for a given DNS name) or service records (when there may be multiple
interchangeable instances).  The `registration` block of the configuration file
determines which records are created.  This block contains properties:

Property    | Type                     | Description
----------- | ------------------------ | -----------
`"domain"`  | string                   | DNS name under which records will be created for this instance
`"aliases"` | optional array of string | array of fully-qualified DNS names to create as additional host records for this instance
`"type"`    | string                   | the specific subtype of record to use for the host records created for this instance
`"service"` | optional object          | if present, a service record will be created with properties described by this object (see below)

Registrar creates the following records:

* A host record is *always* created at `$(hostname).$domain`.  `$(hostname)`
  here is the system's hostname (see hostname(1)) and `$domain` refers to the
  configuration property above.
* If the `aliases` array is present, then additional host records are created
  for each string in the array.  These should be fully qualified -- they should
  generally end with the value of `domain`.
* If `service` is present, a service record is created at the DNS name `$domain`
  itself.  The `service` object is described below.

All records -- host records and service records -- internally have a specific
`type`.  The `"type"` property above controls the types used for the host
records that Registrar creates.  (Service records always have type `"service"`,
and any type other than `"service"` indicates a host record.)  The specific
`"type"` determines exactly how Binder uses them.  The following types are
supported:

Type              | Can be queried directly? | Can be used for Service?
----------------- | ------------------------ | ------------------------
`"db_host"`       | yes                      | no
`"host"`          | yes                      | no
`"load_balancer"` | yes                      | yes
`"moray_host"`    | yes                      | yes
`"ops_host"`      | no                       | yes
`"redis_host"`    | yes                      | yes
`"rr_host"`       | no                       | yes

For types that cannot be queried directly ("ops\_host" and "rr\_host"), if you
query the corresponding DNS name, Binder will behave as though they weren't
there.  This is not generally useful in new components.

For types that cannot be used as a service ("db\_host" and "host"), if these
records are found as child nodes of a "service" record, they will not be
included in the DNS results for the service itself.  This is not generally
useful in new components.

In a simpler world, all host record types could be queried directly (meaning
that when you look up a DNS name that maps to a host record of that type, Binder
answers with the address information in that record), and they could also be
used as backing hosts for a "service" record.  For historical reasons, that's
not true, and it's not easy to change.

**Summary:** The most common case is that each instance of a component is
interchangeable, and clients can talk to any one of them.  In that case, you
should use host records of type `"load_balancer"` and separately configure a
"service" record.  This will cause `"domain"` to be a DNS name that lists all of
the active instances' IP addresses, and `$zonename.$domain` can be used to find
the address of specific instances when that's needed (mostly for debugging).  On
the other hand, if you want to create standalone host records that aren't part
of a logical service, use type `"host"` and do not create an associated service
record.  This is not common.

<!--
    XXX Walk through an example at this point.  Show the configuration file and
    what DNS names are created.  Don't use the one that used to be in the
    documentation because it's wrong.

    Then add a section detailing the format for service records.
    Then add a section explaining the use of ports and SRV records.
-->


## ZooKeeper data format

The service discovery information in ZooKeeper is always written by Registrar
and read by Binder.  It's thus a contract between these components.  However, it
was historically not documented, and several pieces are redundant or confusing.
Additionally, this information is not thoroughly validated in Binder.

**Caveat:** This information is provided for reference only.  The existing
implementation is not crisp enough or validated enough to use this information
to write an alternate implementation and expect that it will interoperate with
the existing one.

**Before reading this section, be sure to read and understand the "Configuration
reference" section.  It covers the basic underlying concepts that are used in
this section.**


### ZooKeeper paths

ZooKeeper provides a filesystem-like API: there's a hierarchical,
slash-delimited namespace of objects.  Data about DNS domains is stored into
ZooKeeper in paths derived from the domains by reversing the components of the
domain and replacing dots (".") with slashes ("/").  So the information for
domain "authcache.emy-10.joyent.us" is contained under
"/us/joyent/emy-10/authcache" in the ZooKeeper namespace.

For a service like authcache, the typical ZooKeeper node structure looks
like this:

* "/us/joyent/emy-10/authcache" contains the service record for "authcache".
* Nodes underneath this path (like
  "/us/joyent/emy-10/authcache/a2674d3b-a9c4-46bc-a835-b6ce21d522c2") contain
  host records for individual instances of "authcache".

The ZooKeeper analog of directory nodes can themselves contain data, so the node
at "/us/joyent/emy-10/authcache" acts as both an object and a directory.

### Overview of service discovery records

All of the ZooKeeper nodes written by Registar contain JSON payloads.  We call
these **service discovery records**.  Internally, every service discovery record
includes:

- required `"type"`, a string identifying the specific type of this record.
- a required property with the same name as the type that provides type-specific
  details, described below.  For example, if the `type` has value `"service"`,
  then there will be a top-level property called `"service"` that contains more
  information.  If the `type` is `"moray_host"`, the top-level property with the
  rest of the details will be called `"moray_host"`.

There are broadly two kinds of records: **host records** and **service
records**.  Host records indicate that a DNS name maps to a particular host
(usually a zone or container).  Service records indicate that a DNS name is
served by one or more other hosts that are specified by child nodes in the
ZooKeeper tree.  Binder will reply to DNS requests with information about all of
the hosts that it finds underneath the "service" record.  For details about host
and service records, see the "Configuration reference" above.

Suppose we have a logical service called `authcache` in a deployment that uses
DNS suffix `emy-10.joyent.us`.  If we query Binder for `A` records for
`authcache.emy-10.joyent.us`, we expect to get a list of IP addresses for the
various instances `authcache`.  We expect we can connect to any of these
instances on some well-known port to use the `authcache` service.  How does this
work?

When we query Binder for `authcache.emy-10.joyent.us`, assuming the result is
not cached, Binder fetches the ZooKeeper node at `/us/joyent/emy-10/authcache`.
There, it finds a service record (with `"type" == "service"`):

    {
      "type": "service",
      "service": {
        "type": "service",
        "service": {
          "srvce": "_redis",
          "proto": "_tcp",
          "port": 6379,
          "ttl": 60
        },
        "ttl": 60
      }
    }

Seeing a service record, Binder then _lists_ the children of the ZooKeeper node
"/us/joyent/emy-10/authcache" to find host records for individual instances of
the `authcache` service.  (Remember, ZooKeeper's namespace looks like a
filesystem, but the nodes that you'd think of as directories can themselves also
contain data.  In this case, the _data_ at `/us/joyent/emy-10/authcache` is the
service record.  The child nodes in that directory describe the specific
instances.)  In this example, that includes two instances:

* a2674d3b-a9c4-46bc-a835-b6ce21d522c2
* a4ae094d-da07-4911-94f9-c982dc88f3cc

Binder also fetches the contents of the child nodes.  These records look like
this:

    {
      "type": "redis_host",
      "address": "172.27.10.62",
      "ttl": 30,
      "redis_host": {
        "address": "172.27.10.62",
        "ports": [ 6379 ]
      }
    }

    {
      "type": "redis_host",
      "address": "172.27.10.67",
      "ttl": 30,
      "redis_host": {
        "address": "172.27.10.67",
        "ports": [ 6379 ]
      }
    }

The record includes the IP address and TTLs that will be included in DNS
answers.  In this case, there are two addresses for
"authcache.emy-10.joyent.us":

    $ dig +nocmd +nocomments +noquestion +nostats authcache.emy-10.joyent.us 
    authcache.emy-10.joyent.us. 30  IN      A       172.27.10.67
    authcache.emy-10.joyent.us. 30  IN      A       172.27.10.62

In order to use these, clients need to know the TCP port that the `authcache`
service uses.

Note that clients can also query for the host records directly:

   $ dig +nocmd +nocomments +noquestion +nostats a2674d3b-a9c4-46bc-a835-b6ce21d522c2.authcache.emy-10.joyent.us
    a2674d3b-a9c4-46bc-a835-b6ce21d522c2.authcache.emy-10.joyent.us. 30 IN A 172.27.10.62

In this case, Binder answers the query by fetching the ZooKeeper node
`/us/joyent/emy-10/authcache/a2674d3b-a9c4-46bc-a835-b6ce21d522c2`, finding the
host record there, and producing an "A" record.

When service or host records include port information (as above), Binder can
also answer DNS "SRV queries.  "SRV" answers identify not just IP addresses, but
also TCP ports for multiple servers listening on the same IP address.  These are
used for situations where multiple processes (usually Node programs) are
listening in the same zone.  For example, in modern Triton deployments, the
`moray.emy-10.joyent.us` domain provides SRV records for each of the instances
within each Moray zone:

<!-- XXX example DNS result -->

The SRV record information comes from this service record under
`/us/joyent/emy-10/moray`:

<!-- XXX -->

and these host records under `/us/joyent/emy-10/moray/XXX` <!-- XXX -->

<!-- XXX -->

SRV records allow clients to find all available servers running inside a
container, which allows for more effective load balancing and resiliency than
using a separate load balancer.

### Host record reference

Host records are those with `"type"` other than `"service"`.  These indicate
that the corresponding DNS name represents a single host (usually a zone or
container).

Supported values of `"type"` (and the semantics of each type) are described in
the "Configuration reference" section above.  Those types map exactly to the
`"type"` field in the host record.  There's also a vestigial type called
`"database"` which was historically produced by Manatee, but this type of record
is no longer used nor consumed.

The various types of host records largely function the same way: each of these
records causes Binder to produce either one `"A"` record with the IP address of
that instance or multiple `"SRV"` records with the IP address and ports of the
various instances contained inside the zone.

Host records are usually ephemeral nodes in ZooKeeper, which means they are
removed when the corresponding Registrar becomes disconnected from ZooKeeper.

In addition to the `"type"` field, each host record these has a top-level
property with the same name as the `"type"` property's value.  For example, a
`"load_balancer"` record has a top-level object called `"load_balancer"`.  This
record includes:

- required `"address"`: an IPv4 address.  This is used to fill in both DNS "A"
  records and "SRV" records.  This should be the IP address on which this
  instance has TCP servers listening.
- optional `"ttl"`: a positive integer used for the TTL of any DNS answers
  generated from this record.  If this is unspecified, it may be specified by a
  higher-level "service" record (described below) or filled in with a default
  value in Binder.
- optional `"ports"`: an array of positive integers identifying TCP ports on
  which this instance is listening for connections.  If these are specified,
  then Binder will be able to answer SRV queries for this domain as described
  above. <!-- XXX that's not quite true; something else seems to be required -->

Here's an example host record that uses these fields:

<!-- XXX -->

### Service record reference

Service records are those with `"type"` equal to `"service"`.  These indicates
that the corresponding DNS name represents a service with multiple equivalent
instances.  Binder will handle queries for this DNS name by querying ZooKeeper
for all of the child records and then reporting all of the results it finds.
Clients generally try to use one or more of these instances at random.

These ZooKeeper records are generally persistent, not ephemeral, since they
represent information about the DNS name, not transient information about what
instances are currently running.

Here's an example service record:

<!-- XXX -->

Besides the `"type"` property, service records have an extra top-level property
called `"service"` which contains any additional information.  This object
itself has a property called `"type"` which is always `"service"`, an optional
`"ttl"` similar to that on host records, and an additional object called
`"service"`.  This last object contains information used by Binder to answer SRV
queries:

* `"srvce"`: the SRV record's service name (e.g., `"_http"` or `"_moray"`)
* `"proto"`: the SRV record's protocol name (usually `"_tcp"`)
* `"port"`: a positive integer identifying the TCP port of the service.  Note
  that this is overridden by a `"ports"` array on the parent object.
* `"ttl`": an optional integer TTL used for SRV answers.  Unlike `"ports"`, a
  value here overrides a value on the parent object.

DNS SRV records also support weights, but these are not supported by Registrar
or Binder.


## License

Registrar is licensed under the
[Mozilla Public License version 2.0](http://mozilla.org/MPL/2.0/).
