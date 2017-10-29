# HA InfluxDB
> **Author:** Steve Anthony (sma310@lehigh.edu) 
>
> USENIX LISA Lab 2017

----------
[TOC]
## Lab Objective
In this lab you will install and configure HAproxy and influx-relay to convert the InfluxDB installation you built in the "Metrics with Influx/Grafana" lab to be replicated and load balanced, ensuring high availability.

----------
## Suggested Knowledge Prerequisites

 It is highly recommended to take the lab “LISA USENIX Lab 2017 - Metrics with InfluxDB/Grafana” before taking this lab.  If you have already reviewed or completed this lab then you will need the following.
 
 - Familiarity with Linux and working from a command line environment.
 - Familiarity with [VirtualBox](https://www.virtualbox.org/wiki/Downloads) helpful, but not required.

 
 ----------
## Suggested Physical Prerequisites
1. A computer able to run two virtual machine with 512MB of RAM and 1 core each, and a virtual machine with 256MB of RAM and 1 core. You will need to install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) version 4.3+ if you don't already have it.

----------
## Let's Begin!

### Background
This lab utilizes InfluxData's influx-relay project to add replication/HA to the open-source community edition of InfluxDB. The figure below, taken from their [Github page](https://github.com/influxdata/influxdb-relay), describes the logical layout of the service we will be building.
```asciidoc
        ┌─────────────────┐                 
        │writes & queries │  
        └─────────────────┘                 
                 │                          
                 ▼                          
         ┌───────────────┐                  
         │               │                  
┌────────│ Load Balancer │─────────┐        
│        │               │         │        
│        └──────┬─┬──────┘         │        
│               │ │                │        
│               │ │                │        
│        ┌──────┘ └────────┐       │        
│        │ ┌─────────────┐ │       │┌──────┐
│        │ │/write or UDP│ │       ││/query│
│        ▼ └─────────────┘ ▼       │└──────┘
│  ┌──────────┐      ┌──────────┐  │        
│  │ InfluxDB │      │ InfluxDB │  │        
│  │ Relay    │      │ Relay    │  │        
│  └──┬────┬──┘      └────┬──┬──┘  │        
│     │    |              |  │     │        
│     |  ┌─┼──────────────┘  |     │        
│     │  │ └──────────────┐  │     │        
│     ▼  ▼                ▼  ▼     │        
│  ┌──────────┐      ┌──────────┐  │        
│  │          │      │          │  │        
└─▶│ InfluxDB │      │ InfluxDB │◀─┘        
   │          │      │          │           
   └──────────┘      └──────────┘           

```
### Configure Networking for VirtualBox
VirtualBox offers several networking options. In order to make the followup lab easier we will use the same network type for both sections. This lab assumes you've made the initial network configuration changes to use the `NAT Network` network type as described in the "Metrics with InfluxDB/Grafana" lab.

### Prepare to Clone the influxlab VM
In order limit the amount of work we need to duplicate, we will work in the "influxlab" VM as much as possible before cloning our instance and making the final changes and connections.

Before beginning, stop and disable the influxdb and telegraf services in the VM. This will help prevent data drift as we bring the second instance online.

>**Reminder:** Your credentials for all VMs are
>UserID: student
>Password: brainfood!

#### Install and Configure influx-relay
First we install Go and Git so we can build the influx-relay binary. We'll install it in /opt/influx-relay.
```bash
$ sudo mkdir -p /opt/influxdb-relay/etc
$ sudo apt-get install golang git
$ sudo -s
export PATH=/opt/influxdb-relay
go get -u github.com/influxdata/influxdb-relay
```
Next create the configuration file and user account we will use to run the influx-relay service.
```bash
$ sudo cp $GOPATH/src/github.com/influxdata/influxdb-relay/sample.toml $GOPATH/etc/relay.toml
$ sudo ln -s $GOPATH/etc /etc/influxdb-relay
$ sudo adduser --system --no-create-home influxdb-relay
```
Update the `relay.toml` file with the IP of this instance. You can remove the `[[udp]]` section, we won't be using it in this lab.
>**Note:**  In the  `[[http]]` section, `bind-addr` should be `0.0.0.0:9096`, in order to enable the VM to listen on all interfaces. We listen on a separate port as the InfluxDB daemon will listen on `8086`.

Now we write a systemd configuration file for this service in /etc/systemd/system/influxdb-relay.service
```ini
# influxrelay.service
[Unit]
Description=InfluxDB Relay
Wants=network.target network-online.target
After=network.target network-online.target

[Service]
Type=simple
Restart=always

ExecStart=/opt/influxdb-relay/bin/influxdb-relay -config /etc/influxdb-relay/relay.toml

User=influxdb-relay
Group=nogroup

TimeoutSec=300

[Install]
WantedBy=multi-user.target
```
Finally, run the `systemctl daemon-reload` command to finish adding the service.

### Make and Update the Clone
Now that we've completed the actions which would be common to both our influx-relay instances, we can clone our VM.

1. Shutdown the `influxlab` VM.
2. Right click on the VM, and chose `Clone`.
3. Name the clone `influxlab2` and check the box to `reinitialize MAC addresses`.
4. Finally, select the `Linked clone` option and click `Clone`.

Start the `influxlab2` instance and update the hostname in the following locations

 - `/etc/hostname`
 - `/etc/hosts`
 
Update the `/etc/influxdb-relay/relay.toml` file with the name and IP of the cloned instance. A complete sample appears as follows:
```asciidoc
[[http]]
name = "relay-http"
bind-addr = "0.0.0.0:9069"
output = [
	{ name = "influxlab", location = "http://10.0.2.2:8086/write"},
	{ name = "influxlab2", location = "http://10.0.2.3:8086/write"},
]
```
### Configure the Load Balancer with HAProxy
Now that influx-relay is configured to send writes to both InfluxDB instances, we add HAproxy as a load balancer in order to direct traffic either to the influx-relay in case of `/write` requests or to InfluxDB directly when a client makes `/query` request.

Start the `haproxy` instance and log in. Install HAproxy by running the following command:
```bash
$ sudo apt-get install haproxy
```

Configuration for HAproxy is located in `/etc/haproxy/haproxy.cfg`. We'll  be adding `frontend` configuration from requests to InfluxDB `port 8086` and Grafana `port 3000`. 

ACL rules will then direct traffic to the appropriate `backend`, either influx-relay or InfluxDB. A complete sample of configuration to append to `haproxy.cfg` appears as follows:
```asciidoc
frontend http_front
	bind *:80
	mode http
 	stats uri /haproxy?stats
	default_backend grafana_back
frontend influx_front
	bind *:8086
	stats uri /haproxy?stats
	acl influx_write path_beg /write
	use_backend influx_relay if influx_write
	default_backend influx_back

backend influx_relay
	balance roundrobin
	server influxlab 10.0.2.2:9096 check
	server influxlab2 10.0.2.3:9096 check
backend influx_back
	balance roundrobin
	server influxlab 10.0.2.2:8086 check
	server influxlab2 10.0.2.3:8086 check
backend grafana_back
	balance roundrobin
	server influxlab 10.0.2.2:3000	check

```
Restart the HAproxy service `sudo systemctl restart haproxy`.

>**Note:** At this point remember to enable and start the `influxdb` and `telegraf` services on the `influxlab` and `influxlab2` instances.

### Update Port Forwarding in VirtualBox
Now that we have added a load balancer, we must update port forwarding rules we set in the "Metrics with InfluxDB/Grafana" lab so that requests are directly from the VM host to the `haproxy` instance.

1. In the VirtualBox main window, click on `File`, then `Preferences`. 
2. Click on the `Network` section.
3. Edit your `NAT Network`.
4. Click `Port Forwarding`.
5. Update a rule to forward host port `8080` (IP address blank) to guest port `80` (enter the IP of the `haproxy` host).

You should be able to view that status of the HAproxy configuration from the VM host by going to `http://localhost/haproxy?stats`.

### Update Grafana Configuration
Finally, we also must update the configuration for the `Data Source` in Grafana. 

1. Log into Grafana and click the Grafana logo in the upper left.
2. Click `Data Sources`.
3. Change the IP of each of the `internal` and `telegraf` data sources to be the IP of the `haproxy` instance. Remember to click `Save & test` to apply the changes.

## What's Next?
Challenge yourself with one of the other labs available, or investigate sharding your data with influx-relay and HAproxy.
