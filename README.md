# Chihaya

**Note:** The master branch may be in an unstable or even broken state during development.

Chihaya is an open source [BitTorrent tracker] written in [Go].

Differentiating features include:

- Protocol-agnostic middleware
- HTTP and UDP frontends
- IPv4 and IPv6 support
- [YAML] configuration
- Metrics via [Prometheus]

[BitTorrent tracker]: http://en.wikipedia.org/wiki/BitTorrent_tracker
[Go]: https://golang.org
[YAML]: http://yaml.org
[Prometheus]: http://prometheus.io

## Why Chihaya?

Chihaya is built for developers looking to integrate BitTorrent into a preexisting production environment.
Chihaya's pluggable architecture and middleware framework offers a simple and flexible integration point that abstracts the BitTorrent tracker protocols.
The most common use case for Chihaya is integration with the deployment of cloud software.

[OpenBittorrent]: https://openbittorrent.com

### Production Use

#### Facebook

[Facebook] uses BitTorrent to deploy new versions of their software.
In order to optimize the flow of traffic within their datacenters, Chihaya is configured to prefer peers within the same subnet.
Because Facebook organizes their network such that server racks are allocated IP addresses in the same subnet, the vast majority of deployment traffic never impacts the congested areas of their network.

[Facebook]: https://facebook.com

#### CoreOS

[Quay] is a container registry that offers the ability to download containers via BitTorrent in order to speed up large or geographically distant deployments.
Announce URLs from Quay's torrent files contain a [JWT] in order to allow Chihaya to verify that an infohash was approved by the registry.
By verifying the infohash, Quay can be sure that only their content is being shared by their tracker.

[Quay]: https://quay.io
[JWT]: https://jwt.io

## Development

### Getting Started

#### Building from HEAD

In order to compile the project, the [latest stable version of Go] and knowledge of a [working Go environment] are required.

**NOTE:** Building in this fashion will download the latest version of all dependencies, which may have introduced breaking changes.
To produce a build with safe versions of dependencies, follow the instructions for reproducible builds.


```sh
$ mkdir chihaya && export GOPATH=$PWD/chihaya
$ go get -t -u github.com/iost-official/chihaya/...
$ $GOPATH/bin/chihaya --help
```

[latest stable version of Go]: https://golang.org/dl
[working Go environment]: https://golang.org/doc/code.html

#### Reproducible Builds

Reproducible builds are handled by using [dep] to vendor dependencies.

```sh
$ mkdir chihaya && export GOPATH=$PWD/chihaya
$ git clone git@github.com:iost-official/chihaya.git $GOPATH/src/github.com/iost-official/chihaya
$ cd $GOPATH/src/github.com/iost-official/chihaya
$ dep ensure
$ go install github.com/iost-official/chihaya/cmd/...
$ $GOPATH/bin/chihaya --help
```

[dep]: https://github.com/golang/dep


#### Testing

The following will run all tests and benchmarks.
Removing `-bench` will just run unit tests.

```sh
$ go test -bench $(go list ./...)
```

### Contributing

Long-term discussion and bug reports are maintained via [GitHub Issues].
Code review is done via [GitHub Pull Requests].

For more information read [CONTRIBUTING.md].

[GitHub Issues]: https://github.com/iost-official/chihaya/issues
[GitHub Pull Requests]: https://github.com/iost-official/chihaya/pulls
[CONTRIBUTING.md]: https://github.com/iost-official/chihaya/blob/master/CONTRIBUTING.md

### Architecture

```
 +----------------------+
 |  BitTorrent Client   |<--------------+
 +----------------------+               |
             |                          |
             |                          |
             |                          |
+------------v--------------------------+-------------------+-------------------------+
|+----------------------+   +----------------------+frontend|                  chihaya|
||        Parser        |   |        Writer        |        |                         |
|+----------------------+   +----------------------+        |                         |
|            |                          ^                   |                         |
+------------+--------------------------+-------------------+                         |
+------------v--------------------------+-------------------+                         |
|+----------------------+   +----------------------+   logic|                         |
||  PreHook Middleware  |-->|  Response Generator  |<-------|-------------+           |
|+----------------------+   +----------------------+        |             |           |
|                                                           |             |           |
|+----------------------+                                   | +----------------------+|
|| PostHook Middleware  |-----------------------------------|>|       Storage        ||
|+----------------------+                                   | +----------------------+|
|                                                           |                         |
+-----------------------------------------------------------+-------------------------+
```

BitTorrent clients send Announce and Scrape requests to a _Frontend_.
Frontends parse requests and write responses for the particular protocol they implement.
The _TrackerLogic_ interface to is used to generate responses for their requests and optionally perform a task after responding to a client.
A configurable chain of _PreHook_ and _PostHook_ middleware is used to construct an instance of TrackerLogic.
PreHooks are middleware that are executed before the response has been written.
After all PreHooks have executed, any missing response fields that are required are filled by reading out of the configured implementation of the _Storage_ interface.
PostHooks are asynchronous tasks that occur after a response has been delivered to the client.
Request data is written to the storage asynchronously in one of these PostHooks.

