# USENIX LISA Labs 2018
## Getting Started
Welcome to Labs. Each of the labs has a VirtualBox VM for your conviencence. Once you've joined the `LISALABS` network, you can download the files from [our webserver](http://service.lisalabs/virtualbox/).

>**Note:** I've had some difficulty getting port forwarding and VM/Internet connectivity working correctly with the Virtual Box OSE 4.3. I'd recommend using the Oracle packages if you encounter problems.

### Import the VM into VirtualBox

Once you've downloaded the zip file containing the VirtualBox files for the lab you're pursing, unzip them using your preferred method.

Next, open the LABNAME.vbox file, eg. docker.vbox and it will open VirtualBox and import the VM for you. You can start the VM by selecting it and then clicking `Start`.

### Configure Networking for VirtualBox
VirtualBox offers several networking options. In order to make the followup lab easier we will use the same network type for all sections. This will require us to make some initial network configuration changes to use the `NAT Network` network type.

> **Note:** VirtualBox offers two similarly named network types, `NAT` and `NAT Network`.  The default `NAT` network type allows VMs to access the Internet, but not address other VMs. The `NAT Network` network option we're using will allow our VMs to connect both to the Internet and be addressable by other VMs we'll run in the "[HA with InfluxDB](https://github.com/ultramathman/lisalabs/blob/master/influxdb_ha.md)" lab.

We configure the `NAT Network` as follows:

1. In the VirtualBox main window, click on `File`, then `Preferences`. 
2. Click on the `Network` section.
3. Create a `NAT Network` if one does not exist. 

Now change the VM to use the new network.

1. Click on the VM name and then click `Settings`.
2. Click on the `Network` section and change the adapter to be `Attached to` the NATNetwork you created.

### Port Forwarding for SSH
Once the VM has started and you've configured networking, you can install ssh, `apt-get install openssh-server`, and get the instance IP `ip addr show`. Then you can forward port `22` on the guest instance to `2222` on the VM host (your computer) and SSH into the instance.

To enable access to the Grafana webpage from the VM host (ie. your computer) we need to forward a port to VirtualBox which will map it onto the VM port 3000. This is configured as follows:

In the VirtualBox main window, click on File, then Preferences.
1. Click on the Network section.
2. Edit your NAT Network.
3. Click Port Forwarding.
4. Add a rule to forward host port `2222` (IP address blank) to guest port `22` (enter the IP of the guest host).

You can now connect as `student` at `localhost:2222` via SSH on your computer to access the instance more easily.

## Directions for Labs
The directions for this lab set are available at:

- [Metrics with Influx/Grafana](https://github.com/ultramathman/lisalabs/blob/master/influxdb_grafana.md)

- [HA InfluxDB](https://github.com/ultramathman/lisalabs/blob/master/influxdb_ha.md)

- [Docker Basics](https://github.com/ultramathman/lisalabs/blob/master/docker.md)

- [Saltstack for Configuration Management](https://github.com/ultramathman/lisalabs/blob/master/saltstack.md)



