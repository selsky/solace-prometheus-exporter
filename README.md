
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-v2.0%20adopted-ff69b4.svg)](CODE_OF_CONDUCT.md)

# solace-prometheus-exporter, a Prometheus Exporter for Solace Message Brokers

## Overview

![Archtiecture overview](https://raw.githubusercontent.com/solacecommunity/solace-prometheus-exporter/master/doc/architecture_001.png)

The exporter is written in go, based on the Solace Legacy SEMP protocol.  
I graps metrics via SEMP v1 and provide those as prometheus friendly http endpoints.


Video Intro available on youtube: [Integrating Prometheus and Grafana with Solace PubSub+ | Solace Community Lightning Talk
](https://youtu.be/72Wz5rrStAU?t=35)

## Features

It implements the following endpoints:  
```
http://<host>:<port>/         Document page showing list of endpoints
http://<host>:<port>/metrics             Golang and standard Prometheus metrics
http://<host>:<port>/solace-std          Deprecated: Solace metrics for System and VPN levels
http://<host>:<port>/solace-det          Deprecated: Solace metrics for Messaging Clients and Queues
http://<host>:<port>/solace-broker-std   Deprecated: Solace Broker only Standard Metrics (System)
http://<host>:<port>/solace-vpn-std      Deprecated: Solace Vpn only Standard Metrics (VPN), available to non-global access right admins
http://<host>:<port>/solace-vpn-stats    Deprecated: Solace Vpn only Statistics Metrics (VPN), available to non-global access right admins
http://<host>:<port>/solace-vpn-det      Deprecated: Solace Vpn only Detailed Metrics (VPN), available to non-global access right admins
http://<host>:<port>/solace              The modular endpoint
```

### Modular endpoint explained

Configure the data you want ot receive, via HTTP GET parameters.
Please use in the format: "m.ClientStats=*|*&m.VpnStats=*|*"  
Here is "m." the prefix.  
Here is "ClientStats" the scrape target.  
The first asterisk the VPN filter, and the second asterisk the item filter. Not all scrape targets support filter.
Scrape targets:


| scape target	| vpn filter supports | item filter supported | performance impact |
|---------------|---------------------|-----------------------|------------------------|
| Version | no | no | dont harm broker |
| Health | no | no | dont harm broker |
| Spool | no | no | dont harm broker |
| Redundancy (only for HA broker) | no | no | dont harm broker |
| ConfigSyncRouter (only for HA broker) | no | no | dont harm broker |
| Vpn | yes | no | dont harm broker |
| VpnReplication | yes | no | dont harm broker |
| ConfigSyncVpn (only for HA broker) | yes | no | dont harm broker |
| Bridge | yes | yes | dont harm broker |
| VpnSpool | yes | no | dont harm broker |
| ClientStats | yes | no | may harm broker if many clients |
| VpnStats | yes | no | has a very small performance down site |
| BridgeStats | yes | yes | has a very small performance down site |
| QueueRates | yes | yes | may harm broker if many queues |
| QueueUsage | yes | yes | may harm broker if many queues |

### Port registration

The [registered](https://github.com/prometheus/prometheus/wiki/Default-port-allocations) default port for Solace is 9628  

## Usage

```
solace_prometheus_exporter -h
usage: solace_prometheus_exporter [&lt;flags&gt;]

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --log.level=info           Only log messages with the given severity or above. One of: [debug, info, warn, error]
      --log.format=logfmt        Output format of log messages. One of: [logfmt, json]
      --config-file=CONFIG-FILE  Path and name of config file. See sample file solace_prometheus_exporter.ini.</code></pre>
```

The configuration parameters can be placed into a config file or into a set of environment variables or can be given via URL. For Docker you should prefer the environment variable configuration method (see below).  
If the exporter is started with a config file argument then the config file entries have precedence over the environment variables. If a parameter is neither found in URL nor the config file nor in the environment the exporter exits with an error.  

### Config File

```bash
solace_prometheus_exporter --config-file /path/to/config/file.ini
```

Sample config file:
```ini
[solace]
# Address to listen on for web interface and telemetry.
listenAddr=0.0.0.0:9628

# Base URI on which to scrape Solace broker.
scrapeUri=http://localhost:8080

# Basic Auth username for http scrape requests to Solace broker.
username=admin

# Basic Auth password for http scrape requests to Solace broker.
password=admin

# Timeout for http scrape requests to Solace broker.
timeout=5s

# Flag that enables SSL certificate verification for the scrape URI.
sslVerify=false
```

### Environment Variables
Sample environment variables:
```bash
SOLACE_LISTEN_ADDR=0.0.0.0:9628
SOLACE_SCRAPE_URI=http://localhost:8080
SOLACE_USERNAME=admin
SOLACE_PASSWORD=admin
SOLACE_TIMEOUT=5s
SOLACE_SSL_VERIFY=false
```

### URL

You can call:
`https://your_exporter:9628/solace?m.ClientStats=*|*&m.VpnStats=*|*&scrapeURI=https%3A%2F%2Fyour-broker%3A943&username=monitoring&password=monitoring`

This allows you to over write the parameters:
- scrapeURI
- username
- password

This provides you a single exporter for all your on prem broker.

Security: Only use this feature with HTTPS.

#### Sample prometheus config

```prometheus
- job_name: 'solace-std'
  scrape_interval: 15s
  metrics_path: /solace-std
  static_configs:
    - targets:
      - https://USER:PASSWORD@first-broker:943
      - https://USER:PASSWORD@second-broker:943
      - https://USER:PASSWORD@third-broker:943
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: solace-exporter:9628
```

## Build

### Default Build
```bash
cd &lt;some-directory&gt;/solace-prometheus-exporter
go build
```

## Docker

### Build Docker Image

A build Dockerfile is included in the repository.<br/>
This is used to automatically build and push the latest image to the Dockerhub repository [solacecommunity/solace-prometheus-exporter](https://hub.docker.com/r/solacecommunity/solace-prometheus-exporter)

### Run Docker Image

Environment variables are recommended to parameterize the exporter in Docker.<br/>
Put the following parameters, adapted to your situation, into a file on the local host, e.g. env.txt:<br/>
```bash
SOLACE_LISTEN_ADDR=0.0.0.0:9628
SOLACE_SCRAPE_URI=http://localhost:8080
SOLACE_USERNAME=admin
SOLACE_PASSWORD=admin
SOLACE_TIMEOUT=5s
SOLACE_SSL_VERIFY=false
```

Then run
```bash
docker run -d \
 -p 9628:9628 \
 --env-file env.txt \
 --name solace-exporter \
 dabgmx/solace-exporter
```

## Bonus Material

The sub directory **testfiles** contains some sample curl commands and their outputs. This is just fyi and not needed for building.

## Security

Please enshure to run this application only in an secured network or protected by a proxy.  
It may reveald insigts of your application you dont want.  
If you use the feature to pass broker credentials via HTTP body/header. You are forced to run this application within kubernetes/openshift or simular to add a HTTPS layer.

## Resources

For more information try these resources:

- The Solace Developer Portal website at: https://solace.dev
- Ask the [Solace Community](https://solace.community)

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

## Authors

See the list of [contributors](https://github.com/solacecommunity/solace-prometheus-exporter/graphs/contributors) who participated in this project.

## License

See the [LICENSE](LICENSE) file for details.
