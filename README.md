# Carpe Noctem Tactical Operations - ImpactPatrol

This project contains the documentation and technical discussion about the deployment and maintenace of InfluxDB, Grafana and Telegraf. The services are used within the community to monitor the status of the `main` server, send notifications to Discord channels for monitoring alerts and power the `stats.carpenoctem.co` dashboards.

## Current configuration

The services are deployed on the community's `tools` server and run inside Docker containers.

## Steps to reproduce current setup

### Requirements

 1. High-privileged account. Currently, the services have been deployed as `root` user
 2. If deployment has to be done on a Linux machine, intermediate knowledge of the system is recommended :-)

### Steps

#### Keep everything tidy

A dedicated directory is recommended (i.e. `impactpatrol`).

#### Setup Docker containers

Use the `docker-compose.yaml` provided in this repository.

In the same directory as `docker-compose.yaml` create the directories `influxdb/data` and `grafana/data`.

Copy the `grafana.ini` and `influx.conf` files provided in this repository in the same directory as `docker-compose.yaml`.

#### Configure web access

Currently, InfluxDB does not need to be reachable from the internet as the `main` server resides in the same network as the `tools` server.

Grafana dashboards on the other hand need to be publicly accessible. Since the only open ports through the firewall are (presumably) 80/443, a reverse proxy is configured so that HTTP/S requests with destination `stats.carpenoctem.co` are redirected to the local Grafana container (which listens for HTTP/S connections on port `3000`). Any reverse proxy is fine, currently the service is exposed via `haproxy` which needs to have the following entries.

 1. In the `# main frontend which proxys to the backends` section, after `frontend main` add
    ```cfg
        acl host_grafana hdr_dom(host) stats.carpenoctem.co
        use_backend grafana if host_grafana
    ```
 2. In the `#backends` section add 
    ```cfg
        backend grafana
            server      local_httpd 127.0.0.1:3000
    ```

