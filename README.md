# Carpe Noctem Tactical Operations - ImpactPatrol

This project contains the documentation and technical discussion about the deployment and maintenace of InfluxDB, Grafana and Telegraf. The services are used within the community to monitor the status of the `main` server, send notifications to Discord channels for monitoring alerts and power the `stats.carpenoctem.co` dashboards.

## Current configuration

The services are deployed on the community's `tools` server and run inside Docker containers.

Grafana (reachable at `stats.carpenoctem.co`) currently serves two dashboards:

- CNTR Stats, linked to the `cntrgraph` database on InfluxDB, displays metrics collected by CNTR on a per-mission basis
- Main server monitoring, linked to the `telegraf` database on InfluxDB, displays metrics collected by a Telegraf agent installed on the community's `main` server with the purpose of monitoring its hardware resources

The first dashboard is publicly available, while the monitoring one requires a user login (_if qualified_, ask for credentials to either [SSgts](https://github.com/orgs/CntoDev/teams/ssgt) or [R&D](https://github.com/orgs/CntoDev/teams/rnd) manager).

The incoming data from the monitoring of `main` server is set to be stored for 30 days. This can be changed by altering the existing [retention policies](https://docs.influxdata.com/influxdb/v1.8/guides/downsample_and_retain/) enforced in InfluxDB.

The services have a limited amount of resources to prevent the entire `tools` server from running out for other existing services (see [resource option](https://docs.docker.com/compose/compose-file/compose-file-v3/#resources) for docker-compose). InfluxDB is limited to use 100% of 1 CPU core and 1024MB of memory, while Grafana is limited to 1 CPU core and 512MB of memory of the `tools` server.

## Steps to reproduce current setup

### Requirements

1.  High-privileged account. Currently, the services have been deployed as `root` user
2.  If deployment has to be done on a Linux machine, intermediate knowledge of the system is recommended :-)

### Steps

#### Keep everything tidy

A dedicated directory is recommended (i.e. `impactpatrol`).

#### Setup Docker containers

The choice of Docker is due to portability to different systems. It is supported on Linux, Windows and even MacOS thus re-deployment on different systems should be pretty straightforward. First, [Docker Engine](https://docs.docker.com/engine/install/) must be installed as well as [Docker Compose](https://docs.docker.com/compose/install/) which is used to provision all the required services at once and ensuring they are kept running.

Use the `docker-compose.yaml` provided in this repository: copy it into the project folder. Alternatively, just clone this repository and use the created directory as root path for the deployment.

In the same directory as `docker-compose.yaml` create the directories `influxdb/data` and `grafana/data`. Both directories should have permissions set to mode `755` at least.

Copy the `grafana.ini` and `influx.conf` files provided in this repository in the same directory as `docker-compose.yaml`. As before, if you clone the repository they will already be in the right place.

#### Configure web access

Currently, InfluxDB does not need to be reachable from the internet as the `main` server resides in the same network as the `tools` server. If the DB needs to be accessible, make sure the [required ports](https://docs.influxdata.com/influxdb/v1.8/administration/ports/) are open through the network.

Grafana dashboards on the other hand need to be publicly accessible. Since the only open ports through the firewall are 80/443, a reverse proxy is configured so that HTTP/S requests with destination `stats.carpenoctem.co` are redirected to the local Grafana container (which listens for HTTP/S connections on port `3000`). Any reverse proxy is fine, currently the service is exposed via [haproxy](http://www.haproxy.org/) which needs to have the following entries.

1.  In the `# main frontend which proxys to the backends` section, after `frontend main` add
    ```cfg
        acl host_grafana hdr_dom(host) stats.carpenoctem.co
        use_backend grafana if host_grafana
    ```
2.  In the `#backends` section add
    ```cfg
        backend grafana
            server      local_httpd 127.0.0.1:3000
    ```

#### Configure Telegraf agent

Telegraf is an application that collects data from various components or programs on the host that it is running on. The configuration consists of inputs and outputs. For example: an input would be CPU statistics and an output would be pushing that data to an InfluxDB to later display that data in Grafana.

The current Grafana dashboard and Telegraf config assumes that you're running Telegraf on a Windows host.

1. Get the download link for latest Telegraf client from: https://portal.influxdata.com/downloads/ and download it
2. Create a directory for Telegraf: C:\Program Files\Telegraf
3. Unzip the .zip file from the download link into this directory. You should now have a directory with a version number in the end.
4. Go into that directory, move the two files (telegraf.exe, telegraf.conf) back to C:\Program Files\Telegraf and delete the (now empty) Telegraf-directory with a version in its name.
5. Replace telegraf.conf with the telegraf.conf from this git repo. 
6. Edit telegraf.conf and head to line 105: `[[outputs.influxdb]]`, update the variables `urls`, `database`, `username`, `password` so they're correct according to your InfluxDB configuration.
7. Open the Windows command prompt as an Administrator.
8. In the commando prompt, test the config with: `C:\"Program Files"\Telegraf\telegraf.exe --config C:\"Program Files"\Telegraf\telegraf.conf --test`
9. All good? Run: `C:\"Program Files"\Telegraf\telegraf.exe --service install --config C:\"Program Files"\Telegraf\telegraf.conf`
10. When that is done, start the Telegraf Windows service with: `net start telegraf`


#### Migrate existing data

All the data is stored in the `influxdb/data` directory, copy its contents to the new `influxdb/data` directory.
