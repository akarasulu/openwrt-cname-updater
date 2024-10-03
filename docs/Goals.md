# Goals

In my home lab, I have machines dual, triple, and multi-booting. Configuring static IP addresses directly on the host OS is a management nightmare. Instead, static IP addresses are centrally managed through OpenWRT using DHCP reservations.

Still I can't track or remember which OS booted on which machine. Nor should I have to track or remember. My router could easily handle that for me.

>**PRIMARY GOAL**: As machine operating systems come up with different hostnames access them consistently through preferred interfaces by simple machine names without having to lookup which OS just booted and what interface is actually connected.

Secondary subtle goals:

1. **Generalized Presence**: With different OS host and interface names using a `host`-`os`[-wifi] naming scheme (i.e. for `maxwell-manjaro`, `maxwell-manjaro-wifi`, `maxwell-bigsur`, `maxwell-bigsur-wif`) I simply want to know if the machine is up and reachable. Checking if `maxwell` resolves then removes the need to check for presence on all machine OS permutations.
2. **Generalized Access**: Sometimes we don't care which OS is running and which OS we've gotten into until we're inside. This simplifies DevOps configurations, i.e. Ansible roles just need to get in then they detect the OS and perform OS specific tasks. Why deal with all the permutations in the inventory, which might not even be up and running resulting in failures?
3. **Identity Preservation**: While enabling all this, we still want different OS instances to be identified differently since they can run different things and impact other systems which need to be tracked and differentiated. SSH into one OS instance should never presume its host key identity is the same as another multi-booted OS identity.

The only way to do this is to assign each OS its own IP using different lease reservations with spoofed MAC addresses, different client ids (DHCP client option 61), or different hostnames (DHCP client option 12). OpenWrt does not allow us to use the client-id via the UI but you can manually configure it. The best way is to have clients use different MAC addresses (MAC spoofing). There’s much less configuration necessary for this on the DHCP server side. For example, Linux can easily spoof its MAC address so I shifted it by one below:

```text
B0:aa:bb:cc:dd:33 (maxwell-bigsur) → xx.x.xxx.101
B0:cc:cc:cc:dd:34 (maxwell-manjaro) → x.x.xxx.100
```

I created two static leases in the router for each OS host name with these two different MAC addresses. I did the same for the `-wifi` hostnames. It works perfectly. Most Linux distributions with most network managers let’s you do this. Mac OS seems to have some issues but workarounds exist. I no clue about Windows but it has to work too.


See [Best Practices](./Practices.md) explaining why a unique identification scheme is preferred and best for multiple concerns including security.

## Implementation Thoughts

> **WARNING**: Old original ideas before implementing

1. Use [UCI](https://openwrt.org/docs/guide-user/base-system/uci) and not grep/awk commands to query and set cname records.
2. Set `cname` by stripping off OS suffix to get **ONLY** the “machine” name then add the dot domain at the end, keeps mapping files portable across networks
3. UCI lookup cname with `${machine}.${domain}` in `/etc/config/dhcp` db to see if it exists
4. If *add*, or *update* then create the entry with target = `${HOSTNAME}.domain`
5. Delegate `service dnsmasq restart` to the debounce service

### Links

* <https://gist.github.com/jwalanta/53f55d03fcf5265938b64ffd361502d5?permalink_comment_id=3804270>
* <https://forum.openwrt.org/t/how-to-detect-dhcp-assignment-to-send-an-email-on-openwrt/24416/8>
* <https://cweiske.de/tagebuch/i-am-home.htm>
