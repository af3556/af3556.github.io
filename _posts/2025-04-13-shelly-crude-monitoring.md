---
layout: post
title: 'Monitoring a Shelly Device, Crudely'
categories: [Home Automation]
tags: [shelly]
---

A simple question of "how can I monitor whether a Shelly device is up and running?", has a multitude of answers, some simple yet flawed, others less simple and less flawed.

Probably the most involved approach involves enrolling the device in a home automation platform such as Home Assistant and rely on that platform's mechanisms to keep an eye on devices. This would be simple to do (free?) if you already have such a system set up. If this is you, I guess you're done ;-). If not, read on.

Perhaps the simplest approach is to ping the device - this will tell you whether it's on the network. However that doesn't mean it's working - it could be zombified (partially crashed/locked up) or be restarting every 5 minutes; a ping can't tell you much.

A better approach is to poll the Shelly API [`http://${SHELLY}/rpc/Shelly.GetStatus`](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Shelly/#shellygetstatus) to fetch important operational attributes such as uptime and device temperature. The uptime units are undocumented but appear to be in seconds.

## A Shell Implementation

The BASH shell script below is intended to be called from cron on a Linux box. The latest version of this script is at [https://github.com/af3556/shelly/blob/main/snippets/pingshelly.sh](https://github.com/af3556/shelly/blob/main/snippets/pingshelly.sh).

Walking through the script design: at the core is a command pipeline that uses [`curl`](https://www.google.com/search?q=curl+getting+started) to fetch the Shelly device's `GetStatus` and passes that to [`jq`](https://jqlang.org/tutorial/) to extract the uptime and temperature from that as tab-separated values, which are read into shell variables. If either field is out of order a notification is sent via [ntfy.sh](https://ntfy.sh/). Empty values imply the `curl` called failed (e.g. the network or device is down), and if the reported uptime is lower than last time we checked, the device has restarted (or the uptime value has wrapped around, the data type is undocumented though presumably it's 32-bit unsigned so that wrap around will occur after 136 years of uninterrupted operation).

The rest of the script is BASH boilerplate, and frankly is probably a good example of why a Python implementation might be simpler ;-) (sunk cost fallacy FTW). Some notes:

- the curl command is wrapped in a function to enable consistently passing the same core arguments to curl each invocation
- when an error is detected notifications for subsequent errors are suppressed until the error is cleared
- the uptime value read from the Shelly device is recorded to a state file for comparison on the next run
- BASH pipelines create multiple subshells and BASH doesn't have a really great way of passing data back _up_ from those subshells - here we're passing the uptime value back as a string, read into the top level shell's `uptime` value via [command substitution](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html) (`uptime=$(...)`)

```bash
#!/bin/bash
# check a shelly's operational state; track uptime and temperature
# will detect
# - offline state (i.e. failed RPC)
# - reboots (uptime decrease)
# - over-temperature

curl_opts() {
  curl --silent --show-error "$@"
}

# the P4o4PM max ambient is 40C; max. internal temp is unspecified
# observationally internal temps are ~+20 above ambient, and general
# consumer/commercial electronics will start having problems ca. 70C
# bash can't do floating point, so must be integer
MAXTEMP=60
NTFY=https://ntfy.sh/jonsson-pumphouse-loadshed

if (( $# < 1 )); then
  echo "usage: $0 shellyhost" >&2
  exit 1
fi

SHELLY="$1"
STATE_FILE=/var/local/$(basename "$0").$(basename "$SHELLY" .).state
LOG_FILE=${STATE_FILE%.state}.log

# write the uptime and errcount variables to a 'state' file on our way out
trap 'declare -p uptime errcount > "$STATE_FILE"' EXIT

uptime=0
errcount=0
if [[ -f "$STATE_FILE" ]]; then
  # shellcheck source=/dev/null # SC1090
  source "$STATE_FILE"
else
  echo "$0 no prior state"
fi

# pipefail: want curl's exit status; also, the pipe not consuming all of curl's
# output is itself a fault (https://mywiki.wooledge.org/BashPitfalls#pipefail)
# note that pipefail doesn't report the exit status of the _first_ command, it
# reports the rightmost _failed_ command
# i.e. when the read subshell exit's 1, this will propagate up
# if you care about intermediate process exit status, use PIPESTATUS
#
# aside: MAXTEMP is available in the subshell as the latter is a forked copy
# of the parent (if there were an exec involved, export would be required)

set -o pipefail
uptime=$(curl_opts "http://${SHELLY}/rpc/Shelly.GetStatus" |
  jq --raw-output '[.sys.uptime, ."switch:0".temperature.tC] | @tsv' | {
    IFS=$'\t' read -r newuptime temperature remainder

    if [[ -z $newuptime || -z $temperature ]]; then
      echo "$0 failed to read JSON data: [$newuptime, $temperature, $remainder]" >&2
      exit 1  # exit the | {} subshell
    fi

    printf -v t "%.0f" "$temperature"
    if (( t > MAXTEMP )); then
      m="$SHELLY high temperature: $temperature"
      echo "$m" >&2
      m+=" [$HOSTNAME]"
      curl_opts -H "Priority: high" -H "Tags: warning" -d "$m" "$NTFY"
    fi

    if (( newuptime < uptime )); then
      m="$SHELLY restarted? Uptime decreased: expected > $uptime, got $newuptime"
      echo "$m" >&2
      m+=" [$HOSTNAME]"
      curl_opts -d "$m" "$NTFY"
    fi
    # %T: -1 = current time
    printf -v logmsg "%(%F %R)T\t%s\t%s" -1 "$newuptime" "$temperature"
    # echo log to stderr when it's connected to a tty (i.e. not cron)
    if [ -t 2 ]; then
      echo "$logmsg" | tee --append "$LOG_FILE" >&2
    else
      echo "$logmsg" >> "$LOG_FILE"
    fi
    # return new uptime to be written to the state file (we're in a subshell)
    echo "$newuptime"
  })

rc=$?
pipes=( "${PIPESTATUS[@]}" )
if (( rc != 0 )); then
  errcount=$((errcount+1))
  m="$0 failed: ${pipes[*]}"
  echo "$m" >&2
  # ntfy only on the first of a sequence of errors
  if (( errcount == 1 )); then
    m+=" [$HOSTNAME]"
    curl_opts -d "$m" "$NTFY"
  fi
  exit 1
else
  errcount=0
fi
```

## What's In A Name?

While its relatively simple to configure an enduring (e.g. static) IP address on the Shelly device and/or your local router, it's even simpler to just use the device's host name as the target for the API call and not refer to IP addresses at all.

Shelly (gen2+) supports [mDNS](https://kb.shelly.cloud/knowledge-base/kbsa-discovering-shelly-devices-via-mdns) which means that _if_ the device is on your local LAN/VLAN (same subnet) _and_ you're using an OS that supports mDNS (e.g. macOS, most Linux distros, and apparently Windows 11), then you should be able to reach the device at its device ID, e.g.  `shellypro4pm-08f9e0aabbcc.local.` (the name seems to be of the form `shelly$MODEL-$MAC`, where `$MODEL` is the model name and `$MAC` is the lowest of the device's MAC addresses (likely the WiFi interface), both stripped of punctuation). DNS (and mDNS) is case-agnostic, so `ShellyPro4PM` is the same as `shellypro4pm`. mDNS is neat in that, in conjunction with the rest of the [zeroconf](https://en.wikipedia.org/wiki/Zero-configuration_networking) stack it means you don't need a router, DHCP server, etc - a basic network of some devices and a switch will "just work" (usually/maybe).

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

Now, mDNS is intended to as a local network protocol, not routed. If your Shelly device is on a different LAN then mDNS will not work. On to the second approach to naming: your local DHCP server. When the Shelly asks your home router for a DHCP address, it'll provide its name as part of the request. Many home routers will register the device's name in the router's local DNS configuration, often with a private domain name of `.lan`, `.routerlogin.net` and so on. In this situation, you should be able to reach the Shelly device - even if its on a segregated VLAN - via its name. This isn't using mDNS but would have the same result.

This second approach does rely on your local router behaving itself, which they don't always do. If your environment works for mDNS then the `.local` mDNS name may be most reliable.
