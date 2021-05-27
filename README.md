# Arch Network Isolator
The microservice paradigm continues to gain popularity among developers of scalable and fault-tolerant systems. 

While usually being deployed in the cloud, the building blocks of such systems need to be deployed, developed and debugged locally on developer's workstations. The heavy use of containers and container orchestration frameworks such as Kubernetes, OpenShift to name a few drives the need to approximate the properties of cloud infrastructure when making the local deployments on software developers workstations. 

Such an approach requires solving the whole set of problems: having to use the reduced datasets, scaled down configurations, simplified network topologies, having to replace certain building blocks that are parts of the cloud provider infrastructure (such as load balancers e.g.) with open source alternatives (Nginx + HAProxy e.g.) to name a few.

There are many very well-known ways to do this. We at [Alliedium](https://github.com/Alliedium) are using a combination of [Docker](https://www.docker.com/) and [Minikube](https://minikube.sigs.k8s.io/docs/start/) to replicate the container cloud infrastructure. 

All this works for those building blocks that just need to be launched to create a sandbox environment for the one or two microservices that a software developer currently works on (by changing or debugging the source code). 

In such cases a software developer usually runs the microservice component directly on the workstation without any container isolation and it is only natural to expect that particular component will continue to communicate normally with the rest of the sandbox infrastructure still running inside multiple containers. This requires a careful configuration of routing between the host machine and Docker/Minikube including [exposing certain ports](https://www.whitesourcesoftware.com/free-developer-tools/blog/docker-expose-port/) from inside containers to the host machine (the workstation in our case).

Moreover, usually all software developer's workstations are placed in the same local subnet. Some of the building blocks of the system might use IP multicast for auto-discovery (as Apache Ignite nodes in our case). In such cases it is important to make sure that local system deployments on each of software development workstations are isolated on a network level to prevent all sorts of unexpected behaviors and problems.

All of the challenges above becomes clearer if we use [Alliedium AIssistant Cloud app](https://marketplace.atlassian.com/apps/1221352/alliedium-aissistant) as an example.

The app is built as a plugin for Atlassian Jira Cloud and actively uses [Apache Ignite](https://ignite.apache.org/) (along with many other system components &mdash; see the notes of our talk at Apache Ignite Meetup about [Boosting Jira Cloud App Development with Apache Ignite](https://go.gridgain.com/rs/491-TWR-806/images/2021-03-02-virtualmeetup.pdf)) as both a distributed database and as a computational grid.

In the case we make changes/debug the source code of components communicating with Ignite, we have to expose [all the ports necessary for Apache Ignite connectivity](https://dzone.com/articles/a-simple-checklist-for-apache-ignite-beginners) from containers running [Ignite server nodes](https://ignite.apache.org/docs/latest/clustering/clustering). 

In this case Ignite client nodes, running both within the containers and outside of them on the host (software development workstation), are able to connect to all other nodes through a properly configured [discovery mechanism](https://ignite.apache.org/docs/latest/clustering/clustering#discovery-mechanisms). Apache Ignite supports a few different node discovery mechanisms but for the local deployment scenario the natural choice is [TCP/IP Discovery](https://ignite.apache.org/docs/latest/clustering/tcp-ip-discovery) implemented through [DiscoverySPI interface](https://ignite.apache.org/releases/latest/javadoc/org/apache/ignite/spi/discovery/DiscoverySpi.html). 

The latter uses [TcpDiscoveryIPFinder interface](https://ignite.apache.org/releases/latest/javadoc/org/apache/ignite/spi/discovery/tcp/ipfinder/TcpDiscoveryIpFinder.html) allowing to find IPs of all other Ignite nodes to form the cluster. There are several implementations of [TcpDiscoveryIPFinder interface](https://ignite.apache.org/releases/latest/javadoc/org/apache/ignite/spi/discovery/tcp/ipfinder/TcpDiscoveryIpFinder.html) but by default [TcpDiscoveryMulticastIpFinder](https://ignite.apache.org/docs/latest/clustering/tcp-ip-discovery#multicast-ip-finder) based on [Multicast](https://en.wikipedia.org/wiki/Multicast) is used. 

And this is where we risk running into the problem of [Ghost nodes](https://dzone.com/articles/a-simple-checklist-for-apache-ignite-beginners):
> One of the most common problems that many people encountered several times whenever they launched a new Ignite node. You have just executed a single node and encountered that you already have two server Ignite nodes in your topology. Most often, it may happen if you are working on a home office network and one of your colleges also run the Ignite server node at the same time. The fact is that by default, Ignite uses the multicast protocol to discover and communicate with other nodes. During startup, Ignite search for all other nodes that are in the same multicast group and located in the same subnetwork. Moreover, if it does, it tries to connect to the nodes.

Certainly, there is a way to fix this by configuring [static IP](https://ignite.apache.org/docs/latest/clustering/tcp-ip-discovery#static-ip-finder), but we may want to retain the above multicast IP finder. The first reason for this is to make our configuration more production-like, the second is to avoid writing down all the IP addresses and ports we are going to connect to. We expect that it should be possible to configure deployment environment automatically, via the scripts. Otherwise we need to make a strong assumption (which frequently doesn't hold) that each member of software development team has sufficient qualification to set everything up correctly manually. Finally, there may be similar issues with other components of the app backend apart from Ignite nodes and clients. Thus, we can formulate our problem as a) having to isolate different software development workstations from each other and b) providing just enough communication between all components deployed within each isolated workstation.

For this purpose we have created a simple command line Python-based [network isolation tool](https://github.com/Alliedium/arch-network-isolator) that automatically configures iptables and Linux Kernel to achieve the desired level of isolation. The tool is designed to for [Arch Linux](https://archlinux.org/) and [Manjaro Linux](https://manjaro.org/) as we heavily use the latter at [Alliedium](https://github.com/Alliedium) for software development (explaining the choice of Manjaro Linux is a topic for a separate article).

Our tool achieves necessary network isolation by allowing all outbound traffic while disabling all inbound connections except for those we deliberately permit. In addition to that it is

* flexible in configuration
* safe to use (makes a backup of previous iptables configuration, displays warning, asks for user confirmations)
* configures not only rules for the host itself, but also for Docker
* automatically persists all the changes

The tool is [implemented in Python 3](https://github.com/Alliedium/arch-network-isolator/blob/master/config_iptables.py) which is already [preinstalled in Manjaro](https://manjaro.org/features/usercases/scientists/) and uses [config_iptables.sh](https://github.com/Alliedium/arch-network-isolator/blob/master/config_iptables.sh) shell script as an entry point. When the script is run without input arguments it displays the following help and does nothing (to prevent undeliberate changing of system parameters):
```
usage: config_iptables.py [-h] [--profile-file PROFILE_FILE] [--noconfirm] [--nopersist] [--persist] --run

Configure network isolation of localhost by changing system parameters and modifying iptables rules

optional arguments:
  -h, --help            show this help message and exit
  --profile-file PROFILE_FILE, -p PROFILE_FILE
                        Path to profile file (DEFAULT VALUE: default.profile),
                        this file should be simple text file with lines
                        having the following format:

                        <scope> <protocol> <port(s)>

                        here <scope> equals either to one of `localhost', `docker', `all',
                        <protocol> may be given by number or name (e.g. `tcp', `udp')
                        while <port(s)> contains either a single port or a port range
                        given as <start-port>:<end-port> or a comma-separated list of
                        ports and port ranges, but their number should not exceed 15
                        (see multiport module of iptables for details), each port is allowed
                        for external access in INPUT, DOCKER-USER chains or both of them

  --noconfirm           Modify system parameters and iptables rules without confirmation

  --nopersist           Modify system parameters and iptables rules by the way not
                        persisting between boots, may be used for validation,
                        if something goes wrong it is sufficient to reboot to
                        restore previous parameters

  --persist             Modify system parameters and iptables rules with persistence
                        between boots (in the case --noconfirm is passed while
                        --nopersist is not passed, the behavior is the same as if
                        --persist was given)

  --run                 Should be given to proceed with modifications,
                        otherwise help is displayed, this is just for safety

config_iptables.py: error: the following argument is required: --run
```
This is done for safety reasons to prevent undeliberate launching of the tool.

The argument `--profile-file` allows to pass the path to the profile file with additional ports for which external inbound traffic is to be allowed (the single port that is allowed unconditionally is **TCP/22** used for [`SSH`](https://wiki.archlinux.org/index.php/Secure_Shell), moreover, the latter connection is additionally protected from attacks by a rate limit. Each line in this file should contain three space-separated values, namely, **scope**, **protocol** and **port**. The first from them have three possible values:

* `localhost` &mdash; rules are applied only to the host, but not to Docker
* `docker` &mdash; rules are applied only to Docker containers, but not to the host itself
* `all` &mdash; rules are applied both to the host and to Docker containers

The value of **protocol** is usually [`tcp`](https://man.archlinux.org/man/iptables-extensions.8#tcp) or [`udp`](https://man.archlinux.org/man/iptables-extensions.8#udp) (but in fact [any protocols](https://man.archlinux.org/man/iptables-extensions.8#multiport) supported by `iptables` are possible, for instance, [UPD-Lite](https://en.wikipedia.org/wiki/UDP-Lite)).

Finally, the third value, **port**, should be given, as can be seen from help above, either as a single port (passed through `--destination-port` argument of [`iptables`](https://man.archlinux.org/man/iptables-extensions.8)) argument or as several ports (in the format of `--destination-ports` argument of [`multiport`](https://man.archlinux.org/man/iptables-extensions.8#multiport) module).

Default profile is in the file [`default.profile`](https://github.com/Alliedium/arch-network-isolator/blob/master/default.profile) containing the following:

```
localhost tcp 5900
localhost udp 5900
all tcp 18000:18300
```

This means that the port 5900 for [VNC](https://wiki.archlinux.org/index.php/x11vnc) is opened both for TCP and UDP protocols only for the host, but not for Docker containers, while a special port range used for our internal development needs is opened both for the host and for Docker.

The meaning of other input parameters of the tool is clear from the help above. In any case all the information on how to rollback of the changes made is displayed in the log. If `--noconfirm` is not given, then the tool asks confirmation for every step to be done with displaying all the detailed info on what is to be changed and how to rollback these changes.

For example, the very first question is as follows:
```
!!!!!!!!!!!!!!!!!!!!!!!!!! ATTENTION !!!!!!!!!!!!!!!!!!!!!!!!!!
This tool should used very carefully because it modifies system
parameters and iptables, the next steps are in-memory only, so
that all the changes can be revoked by rebooting the computer
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Are you sure you want to continue? [y/N]:
```

When neither `--nopersist` nor `--persist` flags are passed and persistence between boots is to be done by the steps that follow, the question is like
```
!!!!!!!!!!!!!!!!!!!!!!!!!! ATTENTION !!!!!!!!!!!!!!!!!!!!!!!!!!
The next step is to make the modified system parameters and
iptables rules persistent between boots, to do this created are
/etc/sysctl.d/999-net_isolation_123456789.conf setting
necessary system parameters (to rollback these changes you will
need just to delete this file and to reboot the computer) and
/etc/iptables/iptables.rules with all modified iptables rules,
old file /etc/iptables/iptables.rules will be copied to backup
file /etc/iptables/iptables.rules.backup (to rollback these
changes just replace new /etc/iptables/iptables.rules by
/etc/iptables/iptables.rules.backup and reboot the computer)
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Are you sure you really want to make the above changes? [y/N]:
```

Thus, the tool keeps the user informed about the changes that are about to be made to the system. This gives the user a chance to prevent unwanted changes.

Finally, the tools checks some prerequisites before it starts to make changes. It checks, for example, that all the necessary tools are installed and that all firewall services like [firewalld](https://wiki.archlinux.org/index.php/Firewalld) are either not installed or at least disactivated. The tool assumes that nothing except `iptables` service is launched. This helps to avoid conflicts with other [firewall tools](https://wiki.archlinux.org/index.php/Category:Firewalls).

When it comes to Linux Kernel parameters their values may be changed either by echoing into some files in subfolders of `/proc/sys` folder (this is not persistent between boots) or via [sysctl](https://wiki.archlinux.org/index.php/sysctl). The latter tool not only allows to change system parameters at runtime, but also is able to automatically load configuration files with `.conf` extension from certain system folders, namely, `/etc/sysctl.d`, `/run/systctl.d`, `/usr/local/lib/sysctl.d` and `/usr/lib/sysctl.d`, in order of precedence (see [manual](https://man7.org/linux/man-pages/man5/sysctl.d.5.html) for details).

In case the same parameter is changed in more than one file the one with the lexicographically latest name will take precedence. Thus, our tool scans the mentioned folders and automatically creates the configuration file that has the lexicographically latest name among all other configuration files to make sure that all the values of system parameters are really set according to the tool.

Network traffic rules for the host are configured to allow all outbound traffic (configured through `OUTPUT` chain for `filer` table), while inbound traffic (configured through `INPUT` chain for `filter` table) is restricted (see also [documentation on `iptables`](https://wiki.archlinux.org/index.php/iptables) for details, especially sections 'Basic concepts' and 'Chains').

Network traffic regulation for Docker is a bit more nuanced. According to the [official Docker documentation](https://docs.docker.com/network/iptables/) the restriction of inbound traffic to Docker containers should be done through `DOCKER-USER` chain of `filter` table, Docker uses this custom chain along with built-in `FORWARD` chain from `filter` table. See also the paper [Be careful, Docker might be exposing ports to the world](https://www.jeffgeerling.com/blog/2020/be-careful-docker-might-be-exposing-ports-world). But there is a caveat described in one of the [comments](https://www.jeffgeerling.com/comment/14173#comment-14173) to this paper. But if to do all this exactly as stated in the [official Docker documentation](https://docs.docker.com/network/iptables/#restrict-connections-to-the-docker-host), thus restricting connections to Docker containers only to the host and configuring allowed ports analogously as it is done for `INPUT` chain above, then Docker containers lose their connection to the Internet!.. To fix the problem it is necessary to add the following rule as the first one available for `DOCKER-USER` chain:

```
iptables -I DOCKER-USER -m state --state ESTABLISHED,RELATED -j ACCEPT
```
It is important to prohibit only all inbound connections that are in `NEW` state, but we need to accept all connections with `ESTABLISHED` and `RELATED` states. What is said in the [official `iptables` documentation](https://linux.die.net/man/8/iptables):

>NEW &mdash; meaning that the packet has started a new connection, or otherwise associated with a connection which has not seen packets in both directions, and\
>ESTABLISHED &mdash; meaning that the packet is associated with a connection which has seen packets in both directions,\
RELATED &mdash; meaning that the packet is starting a new connection, but is associated with an existing connection, such as an FTP data transfer, or an ICMP error.

That means that if some Docker container can start a new connection and `iptables` should allow responses to arrive back into the container, that is previously initiated and accepted exchanges bypass rule checking not only for `INPUT`, but also for `DOCKER-USER` chain.

To summarize, we have shown that [Arch Network Isolator](https://github.com/Alliedium/arch-network-isolator) combines rich functionality and safety (in terms of both networking and being cautious when making changes to system configurations) while remaining very easy to use.

### Links

* [Apache Ignite](https://ignite.apache.org/)
* [Bhuiyan Shamim, A Simple Checklist for Apache Ignite Beginners](https://dzone.com/articles/a-simple-checklist-for-apache-ignite-beginners)
* [Arch Network Isolator Tool](https://github.com/Alliedium/arch-network-isolator)
* Fast Firewall Setup [repo](https://github.com/ChrisTitusTech/firewallsetup) and [video](https://www.youtube.com/watch?v=qPEA6J9pjG8)
* [iptables in Arch Linux]((https://wiki.archlinux.org/index.php/iptables))
* [sysctl in Arch Linux](https://wiki.archlinux.org/index.php/sysctl)
* [Docker and iptables](https://docs.docker.com/network/iptables/)
* [Geerling Jeff, Be careful, Docker might be exposing ports to the world](https://www.jeffgeerling.com/blog/2020/be-careful-docker-might-be-exposing-ports-world)
* [iptables docs](https://linux.die.net/man/8/iptables)
* [Alliedium AIssistant Cloud app](https://marketplace.atlassian.com/apps/1221352/alliedium-aissistant)
* [Apache Ignite Migration Tool](https://github.com/Alliedium/ignite-migration-tool)
