# Saltstack for Configuration Management
> **Author:** Steve Anthony (sma310@lehigh.edu) 
>
> USENIX LISA Lab 2018
>
>Feedback/Problems? Email the author or [open an issue](https://github.com/ultramathman/lisalabs18/issues)! Pull requests also welcome.

----------
**Table of Contents**

- [Lab Objective](#lab-objective)
- [Suggested Knowledge Prerequisites](#suggested-knowledge-prerequisites)
- [Suggested Physical Prerequisites](#suggested-physical-prerequisites)
- [Let's Begin!](#lets-begin)
  - [Install and Configure the Salt Master](#install-and-configure-the-salt-master)
  - [Install and Configure the Salt Minion](#install-and-configure-the-salt-minion)
  - [Targeting Your Minion and Running Commands](#targeting-your-minion-and-running-commands)
  - [Write a State to Install and Manage a Service](#write-a-state-to-install-and-manage-a-service)
  - [Modify the State to Manage a File](#modify-the-state-to-manage-a-file)
  - [Salt Grains and the Pillar](#salt-grains-and-the-pillar)
  - [Template from a Map and Override with a Pillar](#template-from-a-map-and-override-with-a-pillar)
  - [Template Your Configuration with Jinja](#template-your-configuration-with-jinja)
- [What's Next?](#whats-next)
## Lab Objective
In this lab you will install and configure a basic [Saltstack](https://s.saltstack.com/community/) master node, install and configure the Salt minion on a second node, and learn how to select and run commands remotely on that minion.

You will then learn how to use Salt to manage the configuration of files and services on the Minion node from the Master. Finally, you'll learn about salt Grains and the Pillar infrastructure, and how you can use these concepts to help template your configuration definitions and files to allow for more flexible deployment.

----------
## Suggested Knowledge Prerequisites

 - Familiarity with Linux and working from a command line environment.
 - Familiarity with [VirtualBox](https://www.virtualbox.org/wiki/Downloads) helpful, but not required.
 
 ----------
## Suggested Physical Prerequisites
1. A computer able to run a virtual machine with 512MB of RAM and 1 core. You will need to install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) version 4.3+ if you don't already have it.

----------
## Let's Begin!

### Install and Configure the Salt Master

Launch the `saltmaster` VM and log in. The account has sudo privileges.
 
> **Credentials:**
>
> Username: student
>
> Password: brainfood!

Next, add the Saltstack key and repository. 
```bash
$ sudo -s
wget -O - https://repo.saltstack.com/py3/debian/9/amd64/latest/SALTSTACK-GPG-KEY.pub | apt-key add -
cat > /etc/apt/sources.list.d/saltstack.list
deb http://repo.saltstack.com/py3/debian/9/amd64/latest stretch main
```
> **Note:** We'll be using the Python 3 version of the latest version of Saltstack. The [Saltstack Package Repo](https://repo.saltstack.com/) provides instruction for other distribution and for pinning to a specific release of minor version of Salt.

Now, update apt and install and configure the salt-master daemon.
```bash
apt-get update
apt-get install salt-master
```
Create a directory for the Salt fileserver and state configuration files, then add a configuration file to tell the master to use this directory:
```bash
mkdir /srv/salt
cat > /etc/salt/master.d/file_roots.conf
file_roots:
  base:
    - /srv/salt
```
> **Note:** The configuration file expects spaces, not tabs. Each level is prefixed by two additional spaces.

Restart the salt-master daemon. Then get and note the IP address (you'll need this later)
```bash
systemctl restart salt-master
ip addr|grep inet
```
> **Note:** We'll assume this master has an IP of 10.0.2.7 for the purposes of this lab.

At this point you will have a functional salt-master daemon ready to accept a minion.

### Install and Configure the Salt Minion
Likewise, in the salt-minion VM, add the repository and install salt-minion. This is the agent connect to the salt-master daemon and apply commands and changes.
```bash
$ sudo -s
wget -O - https://repo.saltstack.com/py3/debian/9/amd64/latest/SALTSTACK-GPG-KEY.pub | apt-key add -
cat > /etc/apt/sources.list.d/saltstack.list
deb http://repo.saltstack.com/py3/debian/9/amd64/latest stretch main
```
Now, update apt and install and configure the salt-master daemon.
```bash
apt-get update
apt-get install salt-minion
```

By default, the minion will look for a host with a hostname of salt. Create a configuration file to tell it to look for the master at the IP you found on the salt-master VM.
```bash
cat > /etc/salt/minion.d/master.conf
master: 10.0.2.7
```
Now restart the salt-minion daemon and return the to salt-master VM.

```bash
systemctl restart salt-minion
```
Salt requires that you accept the minion's public key on the master to secure the connection before you can manage it. On the salt-master VM list the available keys and then accept the salt-minion key.
```bash
salt-key -L
salt-key -a 'salt-minion'
```
> **Note:** You can add all keys using "salt-key -a" or delete keys using the "-d" flag.

### Targeting Your Minion and Running Commands

Salt provides several ways to target on which nodes a command or state update should run. We'll begin by targeting using the minion id (the name of the minion displayed when we accepted it's key), but you can also use more complex methods, such as a predefined list (or nodegroup), a regular expression, or a single or multiple attributes (grains) we'll learn about in the templating section.

For now, let's start by checking the minion's availability using the "test.ping" execution module. On the salt-master VM run the following command:

```bash
 salt 'salt-minion' test.ping
salt-minion:
    True
```

This lets us know that the master was successfully able to contact the minion, and that it received a response. There are many different [execution modules](https://docs.saltstack.com/en/latest/ref/modules/all/index.html) available which allow you to affect different parts of the minion system, including package management, disk management, user management, and specific service management, eg. Varnish, MySQL, Hadoop, etc. The most general of these is the cmdmod module, specifically cmd.run(). This allow us to run arbitrary commands on any or all of our connected minions. For example:

```bash
salt 'salt-minion' cmd.run "cat /etc/hostname"
salt-minion:
    salt-minion
```
Or to update packages:
```bash
salt 'salt-minion' cmd.run "apt-get -y upgrade"
salt-minion:
    Reading package lists...
    Building dependency tree...
    Reading state information...
    Calculating upgrade...
    The following packages have been kept back:
      linux-image-amd64
    0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.
```

Of course, if you're managing multiple distributions it's easier to use the pkg module instead, which will return any changes made:
```bash
salt 'salt-minion' pkg.upgrade
salt-minion:
    ----------
```
### Write a State to Install and Manage a Service
Orchestrating actions on multiple minions is useful, but Salt's configuration management features will allow us to define a desired end state once and then consistently apply it to our minions to fight configuration drift and make it easy to bring new minions into a predefined state.

There's a preconfigured NFS server running on the salt-master VM. Let's write an NFS client state to install and enable AutoFS on the salt-minion VM. During the installation step, we configured the Salt master daemon to store its configuration in /srv/salt. In that directory let's create a subdirectory for the NFS client.

```bash
cd /srv/salt
mkdir nfs-client
cd nfs-client
touch init.sls
```

Salt uses the directory structure as the base for how you will eventually apply states to minions. In our example the state name right now will be "nfs-client". If instead we had created the directory and file "/srv/salt/nfs/client/init.sls", our state would be nfs.client. The init.sls file will hold the configuration and will be omitted from the title of the state. We could also have created a file "/srv/salt/nfs/client.sls" which would **not** be omitted. If we had done this the state would have been nfs.client.

Now edit the init.sls file and add the following configuration to install and enable the AutoFS service.

```asciidoc
rpcbind-service:
  service.running:
    - name: rpcbind
    - enable: True
    - require:
      - pkg: rpcbind
autofs-pkg:
  pkg.installed:
    - name: autofs
nfs-common:
  pkg.installed
rpcbind:
  pkg.installed
autofs-service:
  service.running:
    - name: autofs
    - enable: True
    - require:
      - pkg: autofs-pkg
      - pkg: nfs-common
      - pkg: rpcbind
```
In this file we tell Salt to install the rpcbind, nfs-common, and autofs packages. We then also ensure the rpcbind is running, then ensure the autofs service is running. Things which are important to note here are:

 - Each configuration action requires a globally unique label (among the states being applied). This label is often used as the "name" parameter in the state definition but can be overridden. We installed the "nfs-common" package by name/label, but overrode autofs-pkg with the correct name "autofs".
 - Note the require statement. Salt is both imperative and declarative. We could have listed the actions in the order we wanted them to be applied and omitted the require statements, or list them in any order and specify dependencies as above.
 - The Salt state files use [YAML](http://yaml.org/) as their default renderer. The most important consequence of this for the above state file is that each layer of the file is indented by two additional spaces. Moreover **tabs and spaces are not equivalent**. The Salt [Understanding YAML](https://docs.saltstack.com/en/latest/topics/yaml/) page covers additional considerations.

Now that we have a state file, we create a Top file, named as such because it exists at the top of the Salt configuration directory. This file is used to group machines by name or attribute and define which states are applied to each system. Edit /srv/salt/top.sls and ensure it has the following contents.

```asciidoc
base:
  'salt-minion':
    - nfs-client
```

This tells the Salt master daemon to apply the "nfs-client" state to the minion named "salt-minion". All minions matching the target will have all assigned states in this file applied when we execute the configuration push. Let's do that now by running the following command:

```bash
salt 'salt-minion' state.highstate
```

This will apply all states in the Top file assigned to the minion with the ID "salt-minion". If you would like to see changes which would be made before applying them, use the following command:
```bash
salt 'salt-minion' state.highstate test=true
```
### Modify the State to Manage a File

Now that we have the AutoFS service installed and running, we need to modify our state to deploy files to ensure the AutoFS service is configured correctly. First create a directory in the state to store the files. This isn't required, but it makes managing more complex states easier.
```bash
mkdir /srv/salt/nfs-client/files
```
Now create the /srv/salt/nfs-client/files/auto.master file:
```asciidoc
# AutoFS Master config
/-    /etc/auto.nfs    --timeout=60
```
Similarly, create /srv/salt/nfs-client/files/auto.nfs:
```asciidoc
# Make sure the IP here is the IP of the salt-master VM
/mnt/nfs    -vers=3 10.0.2.7:/mnt
```

Finally, add the following configuration definitions to your init.sls file:
```asciidoc
/etc/auto.master:
  file.managed:
    - source: salt://nfs-client/files/auto.master
    - watch_in:
      - service: autofs-service
/etc/auto.nfs:
  file.managed:
    - source: salt://nfs-client/files/auto.nfs
    - watch_in:
      - service: autofs-service
```
In this file, we tell Salt to manage the /etc/auto.master and /etc/auto.nfs files on the minion using the files we created. Further we introduce the "watch_in" statement. This tells Salt to watch these files for changes, and when changes occur to restart the autofs-service. Recall that this label is for the AutoFS service, so restarting this will enable any changes to take effect immediately.

Apply the changes by running another highstate:

```bash
salt 'salt-minion' state.highstate
```
On the salt-minion VM verify the configuration is working:

```bash
$ cat /mnt/nfs/README.txt 
Congratulations, it works!
```

### Salt Grains and the Pillar

Now we have a configuration definition to install packages, enable a service, deploy configuration files, and restart that service when those files change. We now consider another feature of Salt, useful for targeting minions and in eventually templating state definition to make them more general. The Salt grains interface provides a way for the master to query and use information about the minion system including things such as operating system, IP address, kernel, whether the minion is a virtual or physical machine, and many other properties.

To list the grains which have been collected from the "salt-minion" minion and their data values, we can use the following command:

```bash
salt 'salt-minion' grains.items
```

We can also query a specify grain, eg.
```bash
salt 'salt-minion' grains.get ip_interfaces:lo
salt-minion:
    - 127.0.0.1
    - ::1
```

Instead of targeting minions by name, we can use a grain like so:
```bash
salt -G 'os:Debian' test.ping
```

We can also use grain in our Top file for common configurations:
```asciidoc
base:
  'os:Debian':
    - match: grain
    - nfs-client
```
In addition to grains, which are generated by the minion, Salt also has a master side data tree we can use to send confidential or targeted data to specific minions. Just like with grains, we can list pillar data:
```bash
salt 'salt-minion' pillar.items
```

Notice this returns no data. We have to define it first. Create a directory and Top file for the pillar data:
```bash
mkdir /srv/pillar
touch /srv/pillar/top.sls
```

We'll return to create the actual pillar data in the next section when we template our state.

### Template from a Map and Override with a Pillar

Our state file so far is fine, but inflexible. For example, if we added a CentOS minion, where the package nfs-common doesn't exist and the nfs-utils package serves the same purpose we might duplicate our state file and assign the CentOS style one to those minions and the Debian style on to other minions. Fortunately there's an easier solution. We can define a base set of variables to apply based on a grain, such as os_family, and then override those variable for edge cases using pillars.

Let's create a [Jinja](http://jinja.pocoo.org/) map for our nfs-client state. Create a file /srv/salt/nfs-client/map.jinja:
```asciidoc
{% set nfs = salt['grains.filter_by']({
  'Debian': {
    'pkgs_client': ['nfs-common', 'autofs', 'rpcbind'],
  },
  'RedHat': {
   'pkgs_client': ['nfs-utils', 'autofs', 'rpcbind'],
  }
}, grain='os_family', merge=salt['pillar.get']('nfs:lookup')) %}
```
> **Note:** The last line specifies the grain we're targeting (os_family) and merges this file with a lookup tree in an nfs pillar. While we won't need to override these variables, we can add additional minion specific variables using this same pillar.

We can now import this file in /srv/salt/nfs-common/init.sls and use the pkgs_client variable. We'll also take the opportunity to install both packages in one action. Our new init.sls file looks like this:

```asciidoc
{% from "nfs-client/map.jinja" import nfs with context %}
rpcbind-service:
  service.running:
    - name: rpcbind
    - enable: True
    - require:
      - pkg: nfs-client-pkgs
nfs-client-pkgs:
  pkg.installed:
    - pkgs: {{ nfs.pkgs_client|json }}
autofs-service:
  service.running:
    - name: autofs
    - enable: True
    - require:
      - pkg: nfs-client-pkgs 
/etc/auto.master:
  file.managed:
    - source: salt://nfs-client/files/auto.master
    - watch_in:
      - service: autofs-service
/etc/auto.nfs:
  file.managed:
    - source: salt://nfs-client/files/auto.nfs
    - watch_in:
      - service: autofs-service
```

### Template Your Configuration with Jinja

Finally, let's create a pillar to hold our mount information for autofs and refactor our auto.nfs file so that Salt will generate it for the minion from the grains. First let's create a directory for our pillar. The pillar layout is analogous to the state tree in /srv/salt and the subsequent naming in /srv/salt/top.sls.

```bash
mkdir -p /srv/pillar/nfs
```

Now create /srv/pillar/nfs/salt-minion.sls so that it contains the following, substituting your salt-master VM IP:

```asciidoc
lookup:
  nfs:
    client:
      autofs:
        files:
          auto.nfs:
            /mnt/nfs:
              opts: "-vers=3"
              server: "10.0.2.7"
              path: "/mnt"
```

This file now contains the information we had used in our auto.nfs file. The way it's structured, we could also add other auto.* files to split up the configuration further, or perhaps add a section at the autofs level to also use the file to manage fstab, or add a server branch at the client level for configuration of a host as both an nfs server and client. 

> **Note:** We began our pillar with "lookup:nrpe", so this data structure will be merged with our map.jinja defined earlier. If we had created a pkgs_client label, it would have overwritten the values set in map.jinja.

Next, create /srv/pillar/top.sls and assign the nfs.salt-minion pillar we created to salt-minion.

```asciidoc
base:
  'salt-minion':
    - nfs.salt-minion
```
Verify the pillar has been applied by running the following command again. This time you should see the nfs pillar data in the output.

```bash
salt 'salt-minion' pillar.items
``` 

Now modify /srv/salt/nfs-client/init.sls to tell Salt to expect Jinja in the auto.nfs file. Replace the /etc/auto.nfs block with the following:

```aciidoc
{% for file, contents in salt['pillar.get']('lookup:nfs:client:autofs:files', {}).items() %}
/etc/{{ file }}:
  file.managed:
    - source: salt://nfs-client/files/auto.client
    - template: jinja
    - context:
      file: {{ contents }}
    - watch_in:
      - service: autofs-service
{% endfor %}
```
What we're doing here is iterating over the dictionary we created using the pillar. So for file name in the "files" key in the pillar, Salt will generate a configuration stanza for that file, pointing it at a new template we'll create next at /srv/salt/nfs-client/files/auto.client. Furthermore, we pass the value associated with the "file" key (which is another dictionary containing the mountpoint and options) to the code we'll put in auto.client as a variable called "file". Create the /srv/salt/nfs-client/files/auto.client file with the following contents.

```asciidoc
# File managed by Salt at {{ source }}
# Your changes will be overwritten.
{% for mnt, params in file.iteritems() %}
{% if mnt == 'star' %} 
*    {{ params.opts }}    {{ params.server }}:{{ params.path }}
{% else %}
{{ mnt }}    {{ params.opts }}    {{ params.server }}:{{ params.path }}
{% endif %}
{% endfor %}
```
Here we iterate over the mounts and parameters we passed (the contents of "file" in the snippet above) as the file variable. We then use the mnt, opts, server, and path from the dictionary to build our AutoFS mount configuration.

> **Note:** The "source" variable is passed from the configuration we defined  and will resolve "salt://nfs-client/files/auto.client" in this instance. Even if you don't auto-generate the file, using this Jinja makes it easy to see where the configuration file originated on the minion.

In generating the configuration in this manner, we can avoid duplicating our configuration state file to accommodate many different systems with varying mount requirements. Run a state.highstate to apply the new configuration, and they verify your can still see /mnt/nfs/README.txt on the "salt-minion" VM.

## What's Next?
We have a few options for what to do next.  You can try one of our other labs, you can continue to modify the nfs-client state to add different mounts or remove the whitespace Jinja added to the auto.nfs file on the salt-minion VM, or you can experiment with creating additional configuration states.
