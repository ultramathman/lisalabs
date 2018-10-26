# Metrics with Influx/Grafana
> **Author:** Steve Anthony (sma310@lehigh.edu) 
>
> USENIX LISA Lab 2018
>
>Feedback/Problems? Email the author or [open an issue](https://github.com/ultramathman/lisalabs/issues)! Pull requests also welcome.

----------
**Table of Contents**

- [Lab Objective](#lab-objective)
- [Suggested Knowledge Prerequisites](#suggested-knowledge-prerequisites)
- [Suggested Physical Prerequisites](#suggested-physical-prerequisites)
- [Let's Begin!](#lets-begin)
  - [Install and Configure InfluxDB and the Telegraf Collector](#install-and-cofigure-influxdb-and-the-telegraf-collector)
  - [Install and Configure Grafana](#install-and-configure-grafana)
  - [Enable Access from the VM Host](#enable-access-from-the-vm-host)
  - [Add InfluxDB as a Grafana Data Source](#add-influxdb-as-a-grafana-data-source)
  - [InfluxDB Series Monitoring Dashboard](#influxdb-series-monitoring-dashboard)
  - [Templated Host Dashboard](#templated-host-dashboard)
- [What's Next?](#whats-next)
## Lab Objective
In this lab you will install and configure a basic [InfluxDB](https://docs.influxdata.com/influxdb/v1.6/), [Telegraf](https://docs.influxdata.com/telegraf/v1.8), and [Grafana](http://docs.grafana.org/) installation, enabling you to collect metrics from the host machine and track internal performance of InfluxDB. 

You will also learn how to add InfluxDB as a data source in Grafana, and create basic graphs and dashboards. Finally, you'll take your system dashboard and use the Grafana templating engine to abstract the graphs so that you can use them with multiple systems.

----------
## Suggested Knowledge Prerequisites

 - Familiarity with Linux and working from a command line environment.
 - Familiarity with [VirtualBox](https://www.virtualbox.org/wiki/Downloads) helpful, but not required.
 - Some understanding of the basics of a SQL query language may be helpful, eg. basic SELECT statements.
 
 ----------
## Suggested Physical Prerequisites
1. A computer able to run a virtual machine with 512MB of RAM and 1 core. You will need to install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) version 4.3+ if you don't already have it.

----------
## Let's Begin!

### Install and Configure InfluxDB and the Telegraf Collector

Launch the `influxlab` VM and log in. The account has sudo privileges.
 
> **Credentials:**
>
> Username: student
>
> Password: brainfood!

Next, add the InfluxData key and repository.
```bash
$ curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
$ sudo -s
cat > /etc/apt/sources.list.d/influxdb.list
deb https://repos.influxdata.com/debian stretch stable
```
> **Note:** The InfluxData website provides links to the respective .deb files for their offering, but we elect to use the repository so that feature and security updates may be applied more easily.

Now, update apt and install, enable, and start InfluxDB.
```bash
$ sudo apt-get update
$ sudo apt-get install influxdb
$ sudo systemctl enable influxdb
$ sudo systemctl start influxdb
```
Likewise, install the Telegraf collector. This is the agent which will poll the system for various data points and store them in the Influx database.
```bash
$ sudo apt-get install telegraf
```

By default, Telegraf will collect the following on a 10 second interval:

 - [CPU information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/CPU_README.md)
 - [Disk information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/DISK_README.md#disk-input-plugin)
 - [Disk IO information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/DISK_README.md#diskio-input-plugin)
 - [Kernel information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/KERNEL_README.md)
 - [Memory information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/MEM_README.md)
 - [Process information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/PROCESSES_README.md)
 - [Swap information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/MEM_README.md)
 - [System information](https://github.com/influxdata/telegraf/blob/release-1.7/plugins/inputs/system/SYSTEM_README.md)

There are many [other plugins](https://docs.influxdata.com/telegraf/v1.8/plugins/inputs/) we can add, depending on the software installed and what information we are looking to collect. We'll create a configuration file in **/etc/telegraf/telegraf.d/custom.conf** in order to adjust the interval to collect and flush every 60 seconds and to add the network statistics plugin.

```asciidoc
[agent]
interval="60s"
flush_interval="60s"
[[inputs.net]]
```
Finally, restart the service.
```bash
$ sudo systemctl restart telegraf
```
The system should now be collecting data. Verify this is happening by querying Influx.
```asciidoc
$ influx -execute "show databases;"
name: databases
name
----
_internal
telegraf
$ influx -database telegraf -execute "show measurements;"
name: measurements
name
----
cpu
disk
diskio
kernel
mem
net
processes
swap
system
```
For more information on the InfluxDB query language, see their [documentation](https://docs.influxdata.com/influxdb/v1.6/query_language/).

### Install and Configure Grafana
As we did with InfluxDB, begin by adding the Grafana repository.
```bash
$ curl https://packagecloud.io/gpg.key | sudo apt-key add -
$ sudo cat > /etc/apt/sources.list.d/grafana.list
deb https://packagecloud.io/grafana/stable/debian/ stretch main
```

Now we install the package, then enable and start the service.
```bash
$ sudo apt-get update
$ sudo apt-get install grafana
$ sudo systemctl enable grafana-server
$ sudo systemctl start grafana-server
```
The Grafana service will listen for requests on port 3000. In a production environment, one might consider using Apache or Nginx as a reverse proxy. We omit this configuration as we will be forwarding the port to the VM host in this lab.

### Enable Access from the VM Host
To enable access to the Grafana webpage from the VM host (ie. your computer) we need to forward a port to VirtualBox which will map it onto the VM port 3000. This is configured as follows:

1. In the VirtualBox main window, click on `File`, then `Preferences`. 
2. Click on the `Network` section.
3. Edit your `NAT Network`.
4. Click `Port Forwarding`.
5. Add a rule to forward host port 8080 (IP address blank) to guest port 3000 (enter the IP of the influxlab host).

### Add InfluxDB as a Grafana Data Source
You should now be able to go to http://localhost:8080 and log into Grafana.

>**Credentials:**
>
>Username: admin
>
>Password: admin

Before we can create a dashboard and graphs, we must add our InfluxDB instance as a Grafana datasource. Click `Add data source` to do so and enter the following:

>Name: telegraf (check default)
>
>Type: influx
>
>URL: http://localhost:8086
>
>Database: telegraf
>
>Min time interval: 60s

Create another datasource for the `_internal` database called `internal`. Now we can create dashboards.

### InfluxDB Series Monitoring Dashboard
Now we'll create a new dashboard with a single graph to track the number of series in the Influx database. The *series cardinality* is the number of unique database, measurement, and tag set combinations in an InfluxDB instance.

This number is what we use in a production instance to determine [what size hardware is needed](https://docs.influxdata.com/influxdb/v1.6/guides/hardware_sizing/#general-hardware-guidelines-for-a-single-node) to run reliably. 

1. Click the Grafana logo in the upper left.
2. Select `Dashboads`, the click `New`.
3. Click `Graph` to add a graph to your new dashboard.
4. Click on `Panel Title` and then `Edit`.
5. Change your data source to `internal`.
6. Click `Select Measurement` and choose `database`.
7. In the SELECT line, click on `value` in `field(value)`, choose `numSeries`.
8. Click the `+` next to `mean()` and choose, `Selectors` -> `last`.
9. In the GROUP BY line, click the `+` and choose `tag(database)`.

>**Note**: If you click the menu icon at the end of the FROM line and choose “Toggle edit mode” you should see the following query.
> ```asciidoc
> SELECT last("numSeries") FROM "database" WHERE $timeFilter GROUP BY time($__interval), "database" fill(null)
> ```

Now we can simplify the labels and title and describe our graph.

1. In the `ALIAS BY` line, type `$tag_database`.
2. Click the `General` tab to name the graph. In the `Title` field enter `Number of Series`.
3. In the `Description` field enter:
```markdown
“Number of unique series in the cluster. We track this to compare it to the notes on [hardware sizing](https://docs.influxdata.com/influxdb/v1.3/guides/hardware_sizing/#general-hardware-guidelines-for-a-single-node) to determine when to get a bigger node.” 
```
This will show up as an `i` in the upper left corner of the graph. We recommend describing graphs so that others can get information related to it without needing to ask directly. 

Finally we'll name and save the dashboard.

1. Click the `X` on the right side below the graph to return to the dashboard.
2. Name your dashboard by clicking the gear icon at the top, and selecting `Settings`.
3. Change `Name` to `Internal Performance` and add an `influx` tag.
4. Click the `X` to return to the dashboard.
5. Save the dashboard by selecting the `save` icon at the top of the page.

### Templated Host Dashboard
Now the we have a basic understanding of creating a dashboard and graph, we can explore use of templating in Grafana. Consider a typical use case in which one desires to monitor the same data sets on multiple hosts. Using the Grafana templating engine, we can create one dashboard and change the information displayed based on the host selected.

Create a new dashboard titled `Host Performance`, and add a graph.

1. Name the panel `Disk Usage`.
2. Choose the `disk` measurement.
3. Click the `+` next to the WHERE condition and add `host = influxlab`.
4. Repeat and add `path = /`.
5. Select the `free` field and change to use the `last()` selector.
6. Click the `+` next to `last()` and choose `Aliasing` -> alias. Alias with the field name.
7. Click the `+` next to `last()` and choose `Fields` -> `field`.
8. Repeat for `total` and `used` fields.
9. Change `ALIAS BY` to `$tag_host - $tag_path $col`.

At this point you should see lines for `total`, `used`, and `free`. Now we'll abstract these selections after defining template variables.

1. Click on the gear icon at the top of the page and choose `Variables`.
2. Click `+Add variable` to add a new variable.

> Name: host
>
> Data source: telegraf
>
> Refresh: On Dashboard Load
>
> Query: SHOW TAG VALUES FROM "system" WITH KEY = "host"

Click `Add` to finish adding the variable, then repeat the process to add a `path` variable. This time, select the `Multi-value` and `Include All` options. Finally, click he `X` to the right of `Templating` to close the Templating menu.

Now modify your query to use the template variables.

1. Next to `WHERE` in your query, change `influxlab` to `/^\$host\$/` and `/` to `/^\$path\$/`.

Close the graph editing options. You should now be able to change the path drop down to see disk usage for various paths on influxlab. If you add additional hosts or devices, you’ll be able to select them from the dropdown menus after reloading the webpage.

Feel free to play around adding additional graphs or types of panels if you’d like. Start by clicking `+Add Row` below your existing graph on the dashboard. 

**Remember to save your dashboard!**
## What's Next?
We have a few options for what to do next.  You can try “[LISA USENIX Lab 2018 - HA with InfluxDB](https://github.com/ultramathman/lisalabs/blob/master/influxdb_ha.md)”, you can experiment with your existing setup and build additional graphs or dashboards, or you can explore additional ways to add data to InfluxDB. If you create something you’d like to share, see a lab coordinator for assistance merging your work into the projected rotation.
