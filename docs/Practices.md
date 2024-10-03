# Best Practices

Obviously, this minimal document covers a very narrow scope. Specifically its on how I manage host identities in my home lab environments: host identity like IP, MAC, hostname, and their SSH host identity etc. Some of this may be adapted to be used anywhere I guess. Hope it helps where and how you might use it.

## Machine vs. Host

Resources are limited in home labs so I'm reusing my limited number of machines to do different things for different projects. Almost all lab machines multi-boot, save a few used for permanent static infrastructure. I've got ZFS Boot Menu, xyz netboot, shoelaces, bootp, pxe-boot, ventoy, and more to reboot machines attended or unattended, manually or automated, locally or remotely to any OS I want or could possibly need.

I identify hardware machines, laptops, desktops, servers, whatever with a machine identifier and a machine name. A late 2013 Mac Book Pro I call `maxwell`, that's its machine name, and it has a machine id of `101` (purposefully in IP octet range usually 100-254). On `maxwell`, I'm booting Arch, Manjaro, and Big Sur from its internal drive, but I can use external USB drives or go disk-less to boot anything using a number of different mechanisms.

When these machines come up with different operating systems, even different instances of the same operating system, each are identified uniquely, even if they're not running concurrently.

>With virtualization and bridging, any OS can boot bare metal and run the other installed multi-boot operating systems on the network concurrently. This approach elegantly handles even this corner condition without collision problems.

My `maxwell` machine comes up with multiple operating system instances as `maxwell-bigsur`, `maxwell-arch`, `maxwell-manjaro` and so on. These are their hostnames, and notice they start with their machine names with a dash followed by a short identifier. Here I used a short operating system name but it can be anything. I have wacky names too with more dashes in between but they all start with the machine name.

I assign different IP addresses for each operating system instance that runs using static reservations in DHCP (dnsmasq on OpenWrt) which include their machine id in the second most significant octets depending on the size of the network. I enumerate the most significant octet:

```hosts
maxwell-bigsur        10.1.101.100
maxwell-bigsur-wifi   10.1.101.101
maxwell-manjaro       10.1.101.102
maxwell-manjaro-wifi  10.1.101.103
maxwell-arch          10.1.101.104
maxwell-arch-wifi     10.1.101.105
```

>The last octet is the host-os-interface identifier.

OS instances all have different names, and different IP addresses for every interface on the same network, hence different reservations. OS instances also have different SSH host keys.

As you can see I append `-wifi` to these hostnames if they also have wireless interfaces to the same networks.

## Why?

I do this because identities should not overlap or collide. Because this is the right way, and doing so results in flexibility to do crazy shit and not break things or cause more headaches. It pays itself off over time and its more secure. Just think of something happening on your network, and you cannot tell how or what made that change when it booted if everything is the same. Good luck tracking that one down using your monitoring tools.

>Sounds tedious to do and it is manually, but you can automate all this.

Copying around SSH host keys is a cop out and it gets dangerous. Using the same hostname, essentially the machine name for all operating systems soon results in a lot of host key verification warnings and problems. That's when most people say fuck it and disable host key verification and that's a bad move security wise, especially if you've got anything running in your infrastructure for long periods of time.

Following this approach costs you. You might not know what's running when. You might have to ping to see which variant is up. Sometimes I just want to ssh or vnc to maxwell and don't care. That's when this CNAME Updater comes in. It let's you maintain a CNAME pointing to whatever host is reachable so you can ssh or vnc into it without giving a shit. It also fixes issues with SSH host verification errors if you use SSH canonical hostnames and CNAMEs.

### Proof its right

Who would think other bare metal directly installed multi-boot operating systems could be run as virtual machines? I simply pointed KVM to their partitions and used some magic. I can boot maxwell-arch bare metal, yet run maxwell-manjaro and maxwell-bigsur in virtual machines on maxwell-arch. Crazy right?

Doing things right covered my ass. If I shared identities, IP addresses, MAC addresses, and hostnames then collisions would prevent these machines from coming up and bridging to the network. If it covers this insane corner cases with ease, then you know it can cover every case.

Although I've been doing this for years in the home lab, this corner case made me realize, deep in my gut, that this approach is always the right way to go.

## Summary

My home lab needs to be very flexible to do more with what little I have. Doing so requires dynamically rebooting machines. A fixed CNAME with dynamic targets lessens the overhead of managing the tradeoffs to gain that flexibility.

>I'll add the IPAM tools and other automation packages later that eliminate the management overheads.

Most importantly, security wise, this is the proper way to do things. Flexibility and maintainability is important but security overrides them both.
