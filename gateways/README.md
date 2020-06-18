# TL;DR

```
> docker-compose -f docker-compose.yml -f docker-compose-dc1.yml -f docker-compose-dc2.yml up -d
> curl http://localhost:9090

{
  "name": "web",
  "uri": "/",
  "type": "HTTP",
  "start_time": "2020-06-18T06:51:36.420188",
  "end_time": "2020-06-18T06:51:36.436712",
  "duration": "16.5236ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "payments-dc2",
      "uri": "http://localhost:9091",
      "type": "HTTP",
      "start_time": "2020-06-18T06:51:36.425151",
      "end_time": "2020-06-18T06:51:36.432701",
      "duration": "8.405ms",
      "body": "PAYMENTS V2",
      "upstream_calls": [
        {
          "name": "currency-dc1",
          "uri": "http://localhost:9091",
          "type": "HTTP",
          "start_time": "2020-06-18T06:51:36.427579",
          "end_time": "2020-06-18T06:51:36.427714",
          "duration": "134.5µs",
          "body": "2 USD for 1 GBP",
          "code": 200
        }
      ],
      "code": 200
    }
  ],
  "code": 200
}
```

# Gateways
This demo highlights Consul Service Mesh gateways which allow cross cluster communication between services, for simplicity the demo connects applications direct to the Consul Server. The recommended architecture is to use a local Consul Client on each node.

It consists of the following features:
* Private Network DC1
* Private Network DC2
* WAN Network (Consul Server, Consul Gateway)
* Consul Datacenter DC1 - Primary
* Consul Datacenter DC2 - Secondary, joined to DC1 with WAN federation
* Consul Gateway DC1
* Consul Gateway DC2
* Web frontend (DC1) communicates with API in DC2 via Consul Gateways
* API service (DC2)

To enable connectivity from a service residing in one datacenter to another, a `Service-Resolver` can be used which hints at the route for service resolution.

```
kind = "service-resolver"
name = "api"

redirect {
  service    = "api"
  datacenter = "dc2"
}
```

In addition to this services which would like to leverage mesh gateways need to have this option explicitly declared in their `Service-Defaults`:

```
Kind = "service-defaults"
Name = "api"

Protocol = "http"

MeshGateway = {
  mode = "local"
}
```

All configuration for service definitions and central config can be found in the `service_config` and `central_config` folders. Config for the Consul Server can be found in the `consul_config` folder.

The web application makes an upstream call to the API which resides in a separate datacenter, networks DC1 and DC2 are not connected. Communication between different datacenters must use the WAN.

![](images/gateways.png)

The full flow through the service mesh is as follows:
* Web app makes upstream app via Envoy running at localhost:9091
* Envoy forwards request to Consul Gateway in DC1
* Mesh Gateway in DC1 forwards request to Consul Gateway in DC2 Over WAN
* Mesh Gateway in DC2 forwards request to upstream Envoy for API service
* Envoy sidecar for API service forwards request to API service which is only listening on localhost
* API service receives request and sends response back through same chain
