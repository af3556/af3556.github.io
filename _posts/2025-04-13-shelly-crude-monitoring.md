---
layout: post
title: 'Monitoring a Shelly Device for Uptime and Temperature'
categories: [Home Automation]
tags: [shelly]
---

A simple question of "how can I monitor whether a Shelly device is up and running?", has a multitude of answers, some simple yet flawed, others less simple and less flawed.

Probably the most involved approach involves enrolling the device in a home automation platform such as Home Assistant and rely on that platform's mechanisms to keep an eye on devices. This would be simple to do (free?) if you already have such a system set up. If this is you, I guess you're done ;-). If not, read on.

Perhaps the simplest approach is to ping the device - this will tell you whether it's on the network. However that doesn't mean it's working - it could be zombified (partially crashed/locked up), or be restarting every 5 minutes, or about to melt from over-temperature - a ping can't tell you much.

A better approach is to poll the Shelly API [`http://${SHELLY}/rpc/Shelly.GetStatus`](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Shelly/#shellygetstatus) to fetch important operational attributes such as uptime and device temperature. The uptime units are undocumented but appear to be in seconds.

## A Shell Implementation

The BASH shell script presented here is intended to be called from cron on a Linux box. The latest version of this script is at [https://github.com/af3556/shelly/blob/main/snippets/pingshelly.sh](https://github.com/af3556/shelly/blob/main/snippets/pingshelly.sh), please open that up in another window while we walk through the design. As an aside: this implementation could as easily be in Python or some other language: six of one, half a dozen of the other.

At the script's core is a command pipeline, within a function, that uses [`curl`](https://www.google.com/search?q=curl+getting+started) to fetch the Shelly device's `GetStatus` and passes that JSON to [`jq`](https://jqlang.org/tutorial/) to extract the uptime and temperature as tab-separated values. These are read into shell variables. If either field is out of order an error is raised. The function returns at least two lines: the uptime and temperature, and on error subsequent lines are error messages.

When all is good the script simply logs the time, uptime and temperature (to a file when run from cron, to stdout otherwise). On error, the script notification sent via [ntfy.sh](https://ntfy.sh/).

Empty values imply the `curl` called failed (e.g. the network or device is down), and if the reported uptime is lower than last time we checked, the device has restarted (or the uptime value has wrapped around, the data type is undocumented though presumably it's 32-bit unsigned so that wrap around will occur after 136 years of uninterrupted operation).

The rest of the script is BASH boilerplate. Some notes:

- the curl command is wrapped in a function to enable consistently passing the same core arguments to curl on each invocation
- only the first of a sequence of errors is reported via ntfy.sh
- the uptime value read from the Shelly device is recorded to a state file for comparison on the next run
- BASH pipelines create multiple subshells and BASH doesn't have a really great way of passing data back _up_ from those subshells - here we're passing values back as multiple lines, reading those into an array within the top level shell

## What's In A Name?

While its relatively simple to configure an enduring (e.g. static) IP address on the Shelly device and/or your local router, it's even simpler to just use the device's host name as the target for the API call and not refer to IP addresses at all.

Shelly (gen2+) supports [mDNS](https://kb.shelly.cloud/knowledge-base/kbsa-discovering-shelly-devices-via-mdns) which means that _if_ the device is on your local LAN/VLAN (same subnet) _and_ you're using an OS that supports mDNS (e.g. macOS, most Linux distros, and apparently Windows 11), then you should be able to reach the device at its device ID, e.g.  `shellypro4pm-08f9e0aabbcc.local.` (the name seems to be of the form `shelly$MODEL-$MAC`, where `$MODEL` is the model name and `$MAC` is the lowest of the device's MAC addresses (likely the WiFi interface), both stripped of punctuation). DNS (and mDNS) is case-agnostic, so `ShellyPro4PM` is the same as `shellypro4pm`. mDNS is neat in that, in conjunction with the rest of the [zeroconf](https://en.wikipedia.org/wiki/Zero-configuration_networking) stack, you don't need a router, DHCP server, etc - a basic local network of a switch and some devices will "just work" (usually/maybe).

The device will respond to its name on any interface (WiFi or Ethernet), in the following example the Pro4PM is connected via Ethernet:

```shell
$ ping -c 3 shellypro4pm-08f9e0aabbcc.local.
PING shellypro4pm-08f9e0aabbcc.local. (192.168.1.123) 56(84) bytes of data.
64 bytes from shelly-pumphouse.jonsson.home (192.168.1.123): icmp_seq=1 ttl=255 time=3.87 ms
64 bytes from shelly-pumphouse.jonsson.home (192.168.1.123): icmp_seq=2 ttl=255 time=0.518 ms
64 bytes from shelly-pumphouse.jonsson.home (192.168.1.123): icmp_seq=3 ttl=255 time=0.707 ms

--- shellypro4pm-08f9e0aabbcc.local. ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 0.518/1.699/3.873/1.538 ms
```

Now, mDNS is intended to as a local network protocol, not routed. If your Shelly device is on a different LAN then mDNS will not work (unless you have a [standards-violating router](https://www.google.com/search?q=unifi+forward+mdns)). On to the second approach to naming: your local DHCP server. When the Shelly asks your home router for a DHCP address, it'll provide its name as part of the request. Many home routers will register the device's name in the router's local DNS configuration, often with a private domain name of `.lan`, `.routerlogin.net` and so on. In this situation, you should be able to reach the Shelly device - even if its on a segregated VLAN - via its name. This isn't using mDNS but would have the same result.

This second approach does rely on your local router behaving itself, which they don't always do. If your environment works for mDNS then the `.local` mDNS name may be most reliable.
