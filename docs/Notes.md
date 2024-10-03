# Technical Notes

Mainly gathered while implementing this.

## **BEWARE** of Handler Scripts

The scripts put into `/etc/hotplug.d/dhcp` are invoked by dnsmasq itself as the parent process. On startup, dnsmasq goes through all its leases and invokes these scripts once for each lease, as well as invoking scripts at runtime when individual lease changes occur. Restarting dnsmasq inside these scripts, which are invoked by dnsmasq itself, leads to utter mayhem especially with the train of invocations triggered by restarts. Essentially, a runaway chain reaction of re-invocations occurs and with file locking this gets dangerous requiring the router to be rebooted to recover. Such coupling problems is why the OpenWrt adopted the use of the DBUS notification mechanism, essentially to decouple and avoid these nightmares.

> Obviously there are also severe race conditions. You're stopping and starting the parent process that invoked the child process shell running the script. No bueno !!!

Originally, I thought a less involved, a "softer", daemon restart using unix process signals, i.e. a SIGHUP might prevent handler script invocations: that was not the case. BTW there is a mechanism, however its complicated by the way OpenWrt reconfigures dnsmasq's signals and configuration to make it work in its environment with UCI. Even so, signals to dnsmasq still seem to trigger re-execution of these DHCP scripts at startup without offering fine grained control. There's no clear and simple solution to this problem. Signals don't seem to hold much promise without mucking with the way OpenWrt configures and runs dnsmasq. Messing with those poorly documented and elusive changes could turn into a nasty quagmire.

I also considered using DBUS notifications to decouple but I had no idea if these events are all supported and available. Plus I had to connect to the bus and parse messages from the source.

### The `((cmd)&)&` or `nohup` hack

I found this, `((cmd)&)&` gem, in a turd pile of [StackOverflow](https://stackoverflow.com/questions/27634696/how-to-run-a-script-in-background-linux-openwrt) posts: see `sep`'s response. It just works and you don't need to install the `coreutils-nohup` package and impose a dependency. I tested this beauty running the following script, exiting the ssh session then logging back in to check if it was still alive writing to `/tmp/bgproc`:

```shell
root@gw:~# cat bgproc
#!/bin/bash
while true; do echo foo >> /tmp/bgproc; sleep 10; done
# Ran it like so from remote:
ssh root@gw.appnet '((/root/bgproc)&)&'
ssh root@gw.appnet tail -f /tmp/bgproc
```

I had no clue what this does to the parent child process relationship so investigation was needed and showed what was happening:

```shell
root@gw:~# ps | grep bgproc | grep -v grep
 9481 root      2132 S    {bgproc} /bin/bash /root/bgproc
root@gw:~# ps | grep -E '[[:space:]]1[[:space:]].*'
    1 root      1296 S    /sbin/procd
root@gw:~# grep PPid /proc/9481/task/9481/status
PPid: 1
```

Wow, wow, wow, this is just great! The child process is completely detached from the parent ash, and the grandparent dropbear process. Its parent is procd, the first process. Just to make sure I tested by running the script in the foreground at a shell prompt, and yes the script's parent was ash, the ash process's parent was attached to the dropbear process.

If our DHCP event handler script is a tiny stub invoking the real `cname_updater` script, then the `cname_updater` script could run in the background with procd as its parent, totally detached, and fully decoupled from dnsmasq. The `cname_updater` script can restart dnsmasq without race conditions but chain reactions will still occur on restarts. The script should still not invoke dnsmasq restarts every time.

Obviously another approach is to just use nohup but this method blows it away: (1) no extra weight on limited space, and (2) no extra dependency. If I need OpenWrt scripts running in the background in a simple way this is the goto construct I will use from now on.

I also suspect, if we start bash in the background with `bash -c`, this might get rid of weird behavioral differences seen when running the script from bash verses when dnsmasq invokes it. For sure dnsmasq is invoking the shell in a very strange way shifting the expected behavior.

### Need Debouncing Yet Again

Due to race conditions, restarts of the dnsmasq service **MUST NOT** be directly invoked by DHCP event handling scripts. Instead, such handler scripts **MAY** inform another process to invoke `service dnsmasq restart`. This problem is ameliorated by using the `((cmd)&)&` construct, however, the problem does not stop here.

The series of script invocations, especially at startup, **MUST** complete before dnsmasq is restarted. Basically, activity **SHOULD** quiet down before restarting the dnsmasq service or a runaway cascade could result. A period of quiescence after initial activity, like 10 seconds, before restarting dnsmasq is the best approach. This absorbs unnecessary requests to repeatedly restart the service. Essentially, a [debouncing](https://medium.com/ghostcoder/debounce-vs-throttle-vs-queue-execution-bcde259768) service is needed. This way the storm of invocations can be absorbed and the chain reaction of restarts can be avoided. Luckily we had to do this a few times in the past.

> The situation is getting complicated with the introduction of another service. Why NOT just have the `cname_updater` track a timer to debounce? Implementing a count down timer in an occasionally event invoked script (not running all the time) is a really bad idea. There's no way to provide assurances to executes the command on time since we don't know when these handler scripts run. A background script or better yet a service is needed.

The external debouncing service maintains a resettable count down timer. When set to T seconds, the timer counts down. When it reaches 0, the timer triggers the dnsmasq service to restart. If a new restart request arrives before the timer reaches 0, the timer resets to T seconds again and starts counting down again to 0 and so on. Once there's a T second period of quiet without any request activity, the dnsmasq service is then restarted. This also gives the DHCP event handler scripts enough time to exit and **NOT** hold any locks.

The `cname_updater` script writes `service dnsmasq restart` requests to one end of the unix pipe of a `debounce` instance process which reads the request from the other end of the pipe. The `debounce` process runs in the background all the time and it creates the pipe when it starts. It maintains the counter variable, resets it, and invokes `service dnsmasq restart` when it reaches zero.

## WARNING: OpenWrt dnsmasq and using client identifiers for matching

This is a tiny bit tangential but the problem can bite you in the arse while leaving you scratching your head for hours. This happens when you're configuring reservations and hostnames on OpenWrt's DHCP static reservations page. When creating reservations do not use client identifiers mixed with MAC addresses for the same interface on a machine. I would stick to client identifiers across the board which are easier to configure on all operating systems with less problems or even better just spoof MAC addresses.

So this behavior does not emerge until you're using the same interface across two or more different leases. For the same interface, if one reservations uses a MAC for matching, and other reservations use client identifiers, then the reservation with the MAC will always override your reservations with client identifier matches. You might be able to mess with ordering to fix this but IMHO just stick to all different MAC addresses or different client identifiers for all reservations on the same interface: do not mix.

## Conclusion on Approach

Simply running the update handler in the background, decoupled from dnsmasq does not cut it. Restarts need to be fully decoupled while needless restarts need to be debounced. A separate OpenWrt debounce-service exists independently for this purpose: see the [debounce-service](https://gitea.appnet/openwrt/debounce-service) project.
