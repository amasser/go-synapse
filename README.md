[![Build Status](https://travis-ci.org/blablacar/go-synapse.png?branch=master)](https://travis-ci.org/blablacar/go-synapse)

# Synapse

Synapse is a Service discovery mecanism. It watch servers for services in a backend and report status in a router.
This simplify service communication with backends and allow auto discovery & hot reconfiguration of the communication.
This provide better services discovery and faul-tolerant communication between services

At BlaBlaCar, we use a synapse for each service node that want to communicate with another service and discover those backend nodes (> 2000). [Nerve](https://github.com/blablacar/go-nerve) report node statuses to a Zookeeper and synapse watch it to update a local Hapoxy. All outgoing communication is going through this haproxy.

## Airbnb

Go-Synapse is a go rewrite of Airbnb's [Synapse](https://github.com/airbnb/synapse) with additiional features.

## Installation

Download the latest version on the [release page](https://github.com/blablacar/go-synapse/releases).

Create a configuration file base on the doc or [examples](https://github.com/blablacar/go-synapse/tree/master/examples).

Run with `./synapse synapse-config.yml`

### Building

Just clone the repository and run `./gomake`


## Configuration

It's a YAML file. You can find examples [here](https://github.com/blablacar/go-synapse/tree/master/examples)

Very minimal configuration file with only one service :
```yaml
  - type: haproxy
    configPath: /tmp/hap.config
    reloadCommand: [./examples/haproxy_reload.sh]

```











## Example Migration ##

Let's suppose your Symfony application depends on a MariaDB database instance.
The database.yaml file has the DB host and port hardcoded:

```yaml
production:
  database: mydb
  host: mydb.example.com
  port: 3306
```

You would like to be able to fail over to a different database in case the original dies.
Let's suppose your instance is running in AWS and you're using the tag 'proddb' set to 'true' to indicate the prod DB.
You set up synapse to proxy the DB connection on `localhost:3306` in the `synapse.conf.json` file.
Add an array under `services` that looks like this:

```json
 "services": [
  {
   "name": "proddb",
   "default_servers": {
     "name": "default-db",
     "host": "mydb.example.com",
     "port": 3306
   },
   "discovery": {
    "method": "zookeeper",
    "path": "/services/proddb",
    "hosts": ["zkhost1","zkhost2"]
   },
   "haproxy": {
    "port": 3307,
    "server_options": "check inter 2000 rise 3 fall 2",
    "frontend": "mode tcp",
    "backend": "mode tcp"
   }
  }
 ]
```

And then change your database.yaml file to look like this:

```yaml
production:
  database: mydb
  host: localhost
  port: 3307
```

Start up synapse.
It will configure HAProxy with a proxy from `localhost:3307` to your DB.
It will attempt to find the DB using the Zookeeper Watcher; if that does not work, it will default to the DB given in `default_servers`.
In the worst case, if Zookeeper is down and you need to change which DB your application talks to, simply edit the `synapse.conf.json` file, update the `default_servers` and restart synapse.
HAProxy will be transparently reloaded, and your application will keep running without a hiccup.

## Installation ##

### Pre-requisite ###
Verify that you have a decent installation of the Golang compiler, you need one.
Then, we use here the [GOM](https://github.com/mattn/gom) tool to manage dependencies and build the synapse binary. All install information can be found on the github repository:
https://github.com/mattn/gom

Optionnaly, you can also install a GNU Make on your system. It's not needed, but will ease the build and install process.

### Build ###

Clone the repository where you want to have it:

git clone https://github.com/blablacar/go-synapse

Install in _vendor directory all dependencies (for a list take a look at the Gomfile):

	gom install

Then you can build the Synapse Binary:

	gom build synapse/synapse

### Makefile ###
If you have a GNU Make or equivalent on your system, you can also use it to build and install nerve.

	`make dep-install` # Will install all go dependencies into _vendor directory

	`make build` # Will compile nerve binary and push it into local bin/ directory
	
	`make test` # Will execute all units tests

	`make install` # Will install nerve binary in the system directory /usr/local/bin (can be overriden at the top of the Makefile)

	`make clean` # Will remove all existing binary in bin/ and remove the dependencies directory _vendor

	`make all` # an alias to make clean dep-install build

### HAProxy ###

Don't forget to install HAProxy too.

## Configuration ##

Synapse depends on a single config file in JSON format; it's usually called `synapse.conf.json`.
The file has four main sections.

1. `instance_id`: the name synapse will use in log; makes debugging easier when using a central log aggregation tool like ELK
2. `log-level`: The log level (any valid value from DEBUG, INFO, WARN, FATAL) (default to 'WARN')
3. [`services`](#services): lists the services you'd like to connect.
4. [`haproxy`](#haproxy): specifies how to configure and interact with HAProxy.

<a name="services"/>
### Configuring a Service ###

The `services` section is a hash, where the keys are the `name` of the service to be configured.
The name is just a human-readable string; it will be used in logs and notifications.
Each value in the services hash is also a hash, and should contain the following keys:

* [`discovery`](#discovery): how synapse will discover hosts providing this service (see below)
* `default_servers`: the list of default servers providing this service; synapse uses these if no others can be discovered
* [`haproxy`](#haproxysvc): how will the haproxy section for this service be configured

<a name="discovery"/>
#### Service Discovery ####

We've included a number of `discoverys` which provide service discovery.
Put these into the `discovery` section of the service hash, with these options:

##### Base #####

The base discovery is useful in situations where you only want to use the servers in the `default_servers` list.
It has only one option:

* `method`: base

##### Zookeeper #####

This discovery retrieves a list of servers from zookeeper.
It takes the following mandatory arguments:

* `method`: zookeeper
* `path`: the zookeeper path where ephemeral nodes will be created for each available service server
* `hosts`: the list of zookeeper servers to query

The discovery assumes that each node under `path` represents a service server.

#### Listing Default Servers ####

You may list a number of default servers providing a service.
Each hash in that section has the following options:

* `name`: a human-readable name for the default server; must be unique
* `host`: the host or IP address of the server
* `port`: the port where the service runs on the `host`

The `default_servers` list is used only when service discovery returns no servers.
In that case, the service proxy will be created with the servers listed here.
If you do not list any default servers, no proxy will be created.  The
`default_servers` will also be used in addition to discovered servers if the
`keep_default_servers` option is set.

<a name="haproxysvc"/>
#### The `haproxy` Section ####

This section is its own hash, which should contain the following keys:

* `port`: the port (on localhost) where HAProxy will listen for connections to the service. If this is omitted, only a backend stanza (and no listen stanza) will be generated for this service; you'll need to get traffic to your service yourself via the `shared_frontend` or manual frontends in `extra_sections`
* `server_port_override`: the port that discovered servers listen on; you should specify this if your discovery mechanism only discovers names or addresses (like the DNS watcher). If the discovery method discovers a port along with hostnames (like the zookeeper watcher) this option may be left out, but will be used in preference if given.
* `server_options`: the haproxy options for each `server` line of the service in HAProxy config; it may be left out.
* `backend`: additional lines passed to the HAProxy config in the `backend` stanza of this service (or listen section if no shared frontend declared)
* `listen`: these lines will be parsed and placed in the `listen` section;
* `shared_frontend`: optional: haproxy configuration directives for a shared http frontend (see below)

<a name="haproxy"/>
### Configuring HAProxy ###

The top level `haproxy` section of the config file has the following options:

* `reload_command`: the command Synapse will run to reload HAProxy
* `config_file_path`: where Synapse will write the HAProxy config file
* `do_writes`: whether or not the config file will be written (default to `true`)
* `do_reloads`: whether or not Synapse will reload HAProxy (default to `true`)
* `do_socket`: whether or not Synapse will use the HAProxy socket commands to prevent reloads (default to `true`)
* `global`: options listed here will be written into the `global` section of the HAProxy config
* `defaults`: options listed here will be written into the `defaults` section of the HAProxy config
* `extra_sections`: additional, manually-configured `frontend`, `backend`, or `listen` stanzas
* `bind_address`: force HAProxy to listen on this address (default is localhost)
* `shared_frontend`: (OPTIONAL) additional lines passed to the HAProxy config used to configure a shared HTTP frontend (see below)
* `restart_interval`: number of seconds to wait between restarts of haproxy (default: 2)
* `restart_jitter`: percentage, expressed as a float, of jitter to multiply the `restart_interval` by when determining the next
  restart time. Use this to help prevent healthcheck storms when HAProxy restarts. (default: 0.0)
* `state_file_path`: full path on disk (e.g. /tmp/synapse/state.json) for caching haproxy state between reloads.
  If provided, synapse will store recently seen backends at this location and can "remember" backends across both synapse and
  HAProxy restarts. Any backends that are "down" in the reporter but listed in the cache will be put into HAProxy disabled (default: nil)
* `state_file_ttl`: the number of seconds that backends should be kept in the state file cache.
  This only applies if `state_file_path` is provided (default: 86400)

Note that a non-default `bind_address` can be dangerous.
If you configure an `address:port` combination that is already in use on the system, haproxy will fail to start.

Another Usefull Note:
HAProxy reload control heavy depends on system clock. If you adjust your clock, when go-synapse running... You can have more reload than expected, or the opposite (reload waiting for n seconds instead of being each `restart_interval`).


### HAProxy shared HTTP Frontend ###

For HTTP-only services, it is not always necessary or desirable to dedicate a TCP port per service, since HAProxy can route traffic based on host headers.
To support this, the optional `shared_frontend` section can be added to both the `haproxy` section and each indvidual service definition.
Synapse will concatenate them all into a single frontend section in the generated haproxy.cfg file.
You can have more than one shared_frontend, by usong different `name`.
Note that synapse does not assemble the routing ACLs for you; you have to do that yourself based on your needs.
For example:

```json
 "output": {
  "type": "haproxy",
  "shared_frontend": [
  {
   "name": "sharedfront1",
   "content": [
       "bind 127.0.0.1:8081",
       "mode http"
   ]
  }
  ],
  "reload_command": "service haproxy reload",
  "config_file_path": "/etc/haproxy/haproxy.cfg",
  "socket_file_path": "/var/run/haproxy.sock",
  "global": [
   "daemon",
   "user    haproxy",
   "group   haproxy",
   "maxconn 4096",
   "log     127.0.0.1 local2 notice",
   "stats   socket /var/run/haproxy.sock"
  ],
  "defaults": [
   "log      global",
   "balance  roundrobin"
  ]
 }
 "services": [
 {
  "name":"service1",
  "discovery": { 
   "method": "zookeeper",
   "path":  "/nerve/services/service1",
   "hosts": ["0.zookeeper.example.com:2181"]
  },
  "haproxy": {
   "server_options": "check inter 2s rise 3 fall 2",
   "shared_frontend": {
    "name": "sharedfront1",
    "content": [
     "acl is_service1 hdr_dom(host) -i service1.lb.example.com",
     "use_backend service1 if is_service1"
    ],
   },
   "backend": ["mode http"]
  }
 },
 {
  "name": "service2",
  "discovery": {
   "method": "zookeeper",
   "path":  "/nerve/services/service2",
   "hosts": ["0.zookeeper.example.com:2181"]
  },
  "haproxy": {
   "server_options": "check inter 2s rise 3 fall 2",
   "shared_frontend": {
    "name": "sharedfront1",
    "content": [
     "acl is_service1 hdr_dom(host) -i service2.lb.example.com",
     "use_backend service2 if is_service2"
    ],
   },
   "backend": ["mode http"]
  }
 }
 ]
```

This would produce an haproxy.cfg much like the following:

```
backend service1
        mode http
        server server1.example.net:80 server1.example.net:80 check inter 2s rise 3 fall 2

backend service2
        mode http
        server server2.example.net:80 server2.example.net:80 check inter 2s rise 3 fall 2

frontend sharedfront1
        bind 127.0.0.1:8081
        acl is_service1 hdr_dom(host) -i service1.lb
        use_backend service1 if is_service1
        acl is_service2 hdr_dom(host) -i service2.lb
        use_backend service2 if is_service2
```

Non-HTTP backends such as MySQL or RabbitMQ will obviously continue to need their own dedicated ports.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

### Creating a Service Dicovery ###

See the Service Discovery [README](src/synapse/discovery/README.md) for
how to add new Service Discovery.
