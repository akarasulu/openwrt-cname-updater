# OpenWrt CNAME Updater

Maintains a single dynamic DNS CNAME pointing to:

1. multi-boot machines no matter which operating system instance and hostname they booted with last, or
2. preferred redundant (wired/wireless) LAN interfaces of machines connected to the same network

The CNAME `name` field is a constant machine name like `maxwell`: it never changes. The CNAME updater **ONLY** changes the CNAME record's `target` field to reflect the OS instance that's currently running and the most preferred interface to use.

All interfaces and multi-boot operating systems with different hostnames, host keys, and IP addresses are always still reachable through a fixed machine name. I.e. CNAME with name `maxwell` has a target field that always points to the preferred wired interfaces first, and/or the OS that booted last:

* maxwell-bigsur
* maxwell-bigsur-wifi
* maxwell-arch
* maxwell-arch-wifi
* maxwell-manjaro
* maxwell-manjaro-wifi

The CNAME Updater presumes each running OS instance possesses a unique fixed identity on the network. Each OS has its own SSH host key. Every interface of the OS is assigned a unique hostname, and a unique static IP address: even wired and wireless interfaces connecting to the same network have different names and IP addresses assigned. No interface's hostname or IP address collides with any other assignment across multi-booting instances.

## Multi-Boot Machines

Multi-boot machines, booting multiple different operating systems or multiple instances of the same operating system, **SHOULD** always use different hostnames, different IP addresses, and different SSH identities. Using the same hostname, and sharing the same IP results in false man in the middle attack warnings and other host key verification problems with SSH. Disabling host key verification or insecurely sharing (copying) the same private key across many operating systems gets tedious and is insecure. Instead, do it right with different hostnames, ip addresses, dhcp reservations, and use this `CNAME Updater` to manage one constant CNAME pointer no matter how machines boot or which interfaces they use to connect.

See [Best Practices](./docs/Practices.md) explaining why a unique identification scheme is preferred and best for multiple concerns including security.

## Machines with Multiple Interfaces

Another use case is to connect to portable machines whether they're roaming around using WiFi or are stationary docked with wired interfaces. At any one time, none, one, or both interfaces may be up and connected to the network. When no interfaces are available, then the CNAME record disappears.

In the other two cases you want to be able to ssh to `maxwell` and have ssh use whichever interface is up at the moment, whether wireless or wired. If both wired and wireless interfaces are up at the same time, the CNAME will points to the "preferred" interface. For the CNAME Updater, the "preferred" CNAME target is the target value that comes first in the mapping file. Most configure the wired interface first as the preferred interface which usually has more bandwidth and less latency than the wireless interface.

## Installation

The Bash shell needs to be installed on the target router. Use the following command to install bash:

```shell
ssh root@router 'opkg update && opkg install bash'
```

> I know many object to unnecessary executables consuming ROM space, but sorry for not being sorry. I standardize all scripts with bash where ever I have to deal with scripting. The nuances between Bourne, Ash, etc cause problems and headaches wasting my time. If you want to convert it, go right ahead and contribute back. Then again, the bash executable consumes less than a megabyte of space and embedded systems are evolving with more flash space rapidly.

The [OpenWrt debounce-service](https://gitea.appnet/openwrt/debounce-service), needs to be installed. Clone it on any machine with `bash` and `direnv` installed to deploy to your OpenWrt router:

```shell
cd debounce-service
direnv allow
install root@router
```

Do the same for this repository:

```shell
cd cname-updater
direnv allow
install root@router
```

The install script deploys the hook script into `/etc/hotplug.d/dhcp`. It also installs the main handler script within the newly created `/usr/share/cname_updater` directory. Thus the dnsmasq process invokes the minimal ash shell hook script, `/etc/hotplug.d/dhcp/cname_updater_hook`, with `update`, `add`, and `remove` DHCP ACTION events. The hook script then invokes bash to run the main handler script.

>**UPDATE**: Stale CNAMEs were being removed when leases timed out. That could take forever, especially if infinite reservations are used. A new CNAME Reaper service was added to remove stale CNAMEs (meaning those with unreachable targets) even before leases expired. This supports the [Goal](./docs/Goals.md) of presence preservation.

## Configuration

Lastly, a mapping file is installed, `/etc/cname_mappings`, to store CNAME target to alias mappings as two columns:

```text
maxwell-arch maxwell
maxwell-arch-wifi maxwell
maxwell-bigsur maxwell
maxwell-bigsur-wifi maxwell
maxwell-manjaro maxwell
maxwell-manjaro-wifi maxwell
```

There are requirements for multi-interface machine hostnames accessing the same network:

1. The wired interface takes the shortest name which is the base name, or prefix (i.e. `maxwell-manjaro`)
2. The wireless interface takes the longer name with the base name + the `-wifi` suffix (i.e. `maxwell-manjaro-wifi`)

Following this convention allows the `cname_updater` script to preferentially override situations where both interfaces are up together and you want to pick the one for the CNAME to point to. By knowing leases and interfaces are related the behavior is thus:

1. On lease add operations, when both interfaces are up, the CNAME mapping will be overridden to use the preferred mapping (based on CNAME line ordering in the mapping file)
2. On lease update operations, when both interfaces are up, the CNAME mapping is overridden to use the preferred mapping (based on CNAME line ordering in the mapping file)
3. On lease remove operations on lease expiration, if at least one interface is reachable, rather than delete the CNAME record, it is updated to point to the reachable target, otherwise the delete proceeds.

Change the mappings to suite your multi-boot and multi-interfaced hosts' needs. You can add as many mappings for the same CNAME as you like. The order impacts preference pointing the CNAME to the preferred hostname. For example when booted into Big Sur the CNAME points to the wired `maxwell-bigsur` even if both it and `maxwell-bigsur-wifi` are actively connected and reachable. Swapping the line order, will prefer WiFi instead.

> **NOTE**: The first column is the target, the second is the CNAME record's alias. When creating cname records, the router's primary domain name is dot appended. Avoiding domains in the mapping file make them reusable across domains and routers.

CNAME pointers provide perfect indirection in these situations avoiding false man in the middle attack warnings or host key verification problems. To do so update your SSH config to use canonical hostnames:

```config
Host *
        CanonicalizeHostname yes
        CanonicalDomains mydomain myotherdomain
        CanonicalizeMaxDots 3
        CanonicalizePermittedCNAMEs *.mydomain:*.mydomain,*.myotherdomain:*.myotherdomain
```

SSH chases the CNAME to find the reachable A record, which it resolves to the host IP, then SSH connects into the machine's IP using the preferred interface.

## Why do this?

Machines in my home lab constantly come up and go down using different multi-boot configurations. Sometimes this happens automatically on-demand or manually using various tools like ZFS Boot Menu, XYZ Netboot, Shoelaces, Ventoy, PXE booting etc.

To avoid tedious man in the middle attack warnings and host key verification problems while doing things right, I use different reservations, hostnames, IP addresses, and SSH host identities for all multi-boot machine operating systems. Many of my machines are laptops which dock on occasion to use better wired connections. The CNAME Updater helps in both use cases.

## Other Stuff

* [ ] @TODO: Switch to UCI conf instead of mappings or a conf file? Probably not worth it
* [ ] @TODO: Give DBUS a try to remove the debounce service dependency. Maybe if DHCP events are supported
* [ ] @TODO: Cleanup those docs, maybe delete them if irrelevant
