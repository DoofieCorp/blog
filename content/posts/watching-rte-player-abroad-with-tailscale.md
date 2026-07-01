+++
categories = ["networking"]
date = "2026-07-01T00:00:00+01:00"
description = "Watching RTE Player from abroad with a Tailscale app connector, and the one missing route that stopped the video from playing."
keywords = ["tailscale", "app connector", "rte", "rte player", "geo-unblock", "openwrt", "wireguard", "subnet router", "streaming", "dns"]
title = "Watching RTE Player abroad with a Tailscale app connector"
+++

I was in a hotel in Portugal and wanted to watch RTE Player. Like most Irish broadcasters it is geo locked to Ireland, and I had no interest in routing my entire phone through a VPN just to get at it. My banking app, my email and everything else could stay on the local connection. I only wanted the traffic that talks to RTE to look like it was coming from home.

I already run [Tailscale](https://tailscale.com/), and my home gateway in Ireland (an OpenWrt box on an ordinary residential line) is both a subnet router and an [app connector](https://tailscale.com/kb/1281/app-connectors). So the plan was tidy. The gateway advertises routes for RTE's CDN ranges, my travel router and phone accept those routes, and only RTE bound traffic hairpins back through Ireland. Split tunnelling, the way it is supposed to work.

It worked for a while, and then it stopped. What follows is the debugging, including a good few wrong turns, because the wrong turns are the interesting part.

## The symptom that misleads you

The RTE Player app loaded perfectly. It showed Irish content, it believed I was in Ireland, the interface was happy. Then I pressed play and the video just spun, and eventually failed.

"The app works but the video does not" is one of the more misleading symptoms in networking, because it sends you looking for exotic problems when the answer is usually mundane. I spent a while refusing to believe it was mundane.

## What it was not

Here is everything that turned out to be innocent, in the order I was sure it was guilty.

First, packet loss. Every SSH session to the gateway was flaky, so obviously the box was dropping packets. I found `udp-broadcast-relay-redux` pinned at 25% of a core in a busy loop, capturing zero SSDP packets while it spun. A restart cleared it, which was satisfying and completely unrelated to the video. Measured loss afterwards was 0% across the LAN, the gateway and the WAN.

Next, MTU. The `tailscale0` interface has a 1280 MTU, and if TCP negotiates a segment size too large for the tunnel, full sized packets get dropped. OpenWrt's `fw4` has an `mtu_fix` option, but it clamps against the route MTU, which for a client to CDN packet is the 1500 byte WAN rather than the 1280 byte return tunnel. So I added a fixed clamp to 1240 on `tailscale0` at both ends. Good hygiene, and the later captures confirmed it working, but it was not the problem either.

Then QUIC. My phone could not get a direct Tailscale connection because the hotel NAT is symmetric, so it was relaying through a DERP server, and QUIC has a habit of misbehaving over relayed paths. I blocked UDP 443 out of the tunnel to force a fall back to HLS over TCP. Toggling that on and off made no difference.

After that, throughput. Surely a relayed tunnel is just slow. I measured it with a `curl` through the tunnel to Cloudflare and got 20 MB in 2.9 seconds, roughly 55 Mbps, which is plenty for a video stream.

Finally I widened the CDN coverage. RTE's video lives on Akamai, and Akamai's address space is enormous, so my first set of ranges missed a chunk of it. Adding comprehensive Akamai coverage helped but did not fix it. I even "simplified" the config at one point by trimming some domains, which promptly broke it again and taught me not to tidy up a system that is currently working.

## Narrowing it down with one question

After all of that the video still would not play in split routing mode, so I asked the question I should have led with. If I set the gateway as a full exit node, does it work?

It did, immediately.

That one fact is worth a lot. If the exit node works but split routing does not, then it is not DRM, not the gateway's address reputation, not throughput and not MTU. It is a coverage gap. Some host that the video needs is not in my advertised routes, so its request leaks out the hotel connection, arrives from a Portuguese address, and gets refused. The exit node works only because it forces everything, including that one host, through Ireland.

So the whole problem reduced to a single question. Which host?

## Finding the missing host

You cannot guess CDN ranges forever, so you have to watch what the device actually asks for. The phone's DNS goes through the travel router's `dnsmasq`, so I turned on query logging, pressed play, let it fail, and read back every hostname the phone looked up. Then, on the gateway, which resolves from Ireland just as the connector does, I resolved each host and checked its addresses against the routes that were actually enabled.

One detail nearly cost me another hour. My first coverage check reported that everything was covered, because `0.0.0.0/0`, the exit node route, was in the enabled list, and every address on earth sits inside `0.0.0.0/0`. Exclude the exit node route from that comparison, or it will happily hide the gap. It is the same reason the exit node "worked": it is the route that swallows everything.

With that corrected, the culprit fell out at once.

## The one hostname

```
link.eu.theplatform.com
```

That is RTE's video entitlement backend, hosted on Comcast's thePlatform. Two things combined against me. It sits on `theplatform.com`, and my connector was only watching `*.theplatform.eu`, one letter off and therefore never discovered. And it load balances across AWS in both Dublin and Frankfurt, on addresses like `18.200.x`, `18.202.x`, `3.76.x`, `52.58.x` and `52.28.x`, none of which were in my ranges.

So the video segments, on Akamai and Cloudflare and CloudFront and AWS Dublin, all downloaded fine over Ireland. But the small entitlement request, the one that decides whether I am allowed to watch, would sometimes resolve to an uncovered address, leak out the hotel connection, reach thePlatform from Portugal and get refused. It was intermittent because it depended on which address DNS handed back. Megabytes of encrypted video had already arrived, but without an entitlement none of it could play.

The app loaded because browsing needs no entitlement. The video failed because it does.

## The fix

Cover the ranges the entitlement host rotates through, in both `autoApprovers` and the advertised routes:

```diff
   "autoApprovers": {
     "routes": {
       // ... LAN subnets + Akamai / Cloudflare / CloudFront / AWS-Dublin
       //     video-CDN blocks already here ...
       "172.224.0.0/12":  ["tag:rte-player"],
+      // theplatform.COM entitlement backend (link.eu.theplatform.com) load
+      // balances across AWS eu-west-1 (Dublin) AND eu-central-1 (Frankfurt);
+      // uncovered ones leaked direct, RTE denied entitlement, no video.
+      "18.200.0.0/13":   ["tag:rte-player"], // AWS Dublin (18.200-18.207)
+      "3.248.0.0/13":    ["tag:rte-player"], // AWS Dublin
+      "3.64.0.0/12":     ["tag:rte-player"], // AWS Frankfurt (incl 3.76)
+      "3.120.0.0/13":    ["tag:rte-player"], // AWS Frankfurt
+      "18.156.0.0/14":   ["tag:rte-player"], // AWS Frankfurt
+      "18.192.0.0/13":   ["tag:rte-player"], // AWS Frankfurt
+      "35.156.0.0/14":   ["tag:rte-player"], // AWS Frankfurt
+      "52.28.0.0/14":    ["tag:rte-player"], // AWS Frankfurt (incl 52.28)
+      "52.56.0.0/14":    ["tag:rte-player"], // AWS Frankfurt (incl 52.58)
+      "89.149.208.0/20": ["tag:rte-player"], // Youbora/NPAW QoS analytics
     },
   },
```

The approver is `tag:rte-player`, the tag the gateway advertises these routes under, so they approve themselves the moment the connector advertises anything inside them. The same ranges also go into the gateway's advertised routes, since `autoApprovers` only decides what gets approved, not what gets advertised.

And the connector's domain list, where the missing letter was the whole bug:

```diff
   "domains": [
     "rte.ie", "*.rte.ie",
     "rasset.ie", "*.rasset.ie",
+    "theplatform.com", "*.theplatform.com",
     "*.theplatform.eu",
     "*.akamai.net", "*.akamaiedge.net",
     "*.akamaihd.net", "*.akamaized.net"
   ],
```

I rechecked every address that had been leaking, confirmed they now route through Ireland, and pressed play. It worked.

## What I would tell myself at the start

A few things would have saved me the afternoon.

If the exit node works and split routing does not, that is a diagnosis and not a workaround. It means a coverage gap and nothing else, so go and find the missing host rather than chasing DRM or throughput.

For a geo locked streamer the video segments are a distraction. The thing that has to appear in country is the entitlement or licence request. Cover that host and the rest follows.

CDN hostnames will lie to you about their domain. This one was `theplatform.com` served from two AWS regions, while I was watching `theplatform.eu`. A wildcard you assume is covering a service quietly is not.

Watch the real DNS. Ten minutes of `dnsmasq` query logging beats hours of guessing at address ranges.

And `0.0.0.0/0` will sabotage your coverage checks, so exclude the exit node route before you conclude that everything is covered.

RTE Player now plays from a hotel in Portugal over a split tunnel, with the rest of the phone's traffic going out the local Wi-Fi, which is exactly the arrangement I wanted.

## Reproduce it yourself

The setup below is RTE specific, but the shape is the same for any geo locked streamer. Only the domains and the CDN ranges change, and the last step tells you how to find yours. You need three things: a Tailscale account, an always on Linux box in the country you want to appear from (mine is an OpenWrt router, but a Raspberry Pi or a spare mini PC on a residential line is fine, and residential matters because streamers flag datacentre addresses), and the client device running Tailscale.

### 1. Declare a tag and an app connector

In the admin console under Access controls, add a tag you own and an app connector that watches the streamer's domains.

```jsonc
"tagOwners": {
  "tag:rte-player": ["your-email@example.com"],
},

"nodeAttrs": [
  {
    "target": ["*"],
    "app": {
      "tailscale.com/app-connectors": [
        {
          "name":       "rte-player",
          "connectors": ["tag:rte-player"],
          "domains": [
            "rte.ie", "*.rte.ie",
            "rasset.ie", "*.rasset.ie",
            "theplatform.com", "*.theplatform.com",   // entitlement, do not omit
            "*.theplatform.eu",
            "*.akamai.net", "*.akamaiedge.net",
            "*.akamaihd.net", "*.akamaized.net"
          ],
        },
      ],
    },
  },
],
```

The connector watches DNS for these domains and advertises routes for whatever addresses they resolve to, which handles the day to day churn. CDNs still load balance widely, so back it with static ranges in the next step.

### 2. Auto approve the CDN ranges

So routes approve themselves instead of needing a manual click every time an address rotates.

```jsonc
"autoApprovers": {
  "routes": {
    // Akamai (video segments)
    "2.16.0.0/13":   ["tag:rte-player"], "2.20.0.0/16":   ["tag:rte-player"],
    "23.0.0.0/12":   ["tag:rte-player"], "23.32.0.0/11":  ["tag:rte-player"],
    "23.64.0.0/10":  ["tag:rte-player"], "23.128.0.0/10": ["tag:rte-player"],
    "23.192.0.0/11": ["tag:rte-player"], "69.192.0.0/16": ["tag:rte-player"],
    "72.246.0.0/15": ["tag:rte-player"], "88.221.0.0/16": ["tag:rte-player"],
    "92.122.0.0/15": ["tag:rte-player"], "95.100.0.0/15": ["tag:rte-player"],
    "96.6.0.0/15":   ["tag:rte-player"], "96.16.0.0/15":  ["tag:rte-player"],
    "104.64.0.0/10": ["tag:rte-player"], "118.214.0.0/16":["tag:rte-player"],
    "172.224.0.0/12":["tag:rte-player"], "173.222.0.0/15":["tag:rte-player"],
    "184.24.0.0/13": ["tag:rte-player"], "184.50.0.0/15": ["tag:rte-player"],
    "184.84.0.0/14": ["tag:rte-player"],
    // Cloudflare (rte.ie front door)
    "104.16.0.0/13": ["tag:rte-player"], "172.64.0.0/13": ["tag:rte-player"],
    // CloudFront
    "3.160.0.0/12":  ["tag:rte-player"], "13.32.0.0/15":  ["tag:rte-player"],
    "13.224.0.0/14": ["tag:rte-player"], "54.192.0.0/16": ["tag:rte-player"],
    "54.230.0.0/16": ["tag:rte-player"],
    // AWS eu-west-1 / Dublin (theplatform feeds, bookmarks)
    "3.248.0.0/13":  ["tag:rte-player"], "18.200.0.0/13": ["tag:rte-player"],
    "34.240.0.0/12": ["tag:rte-player"], "52.16.0.0/13":  ["tag:rte-player"],
    "52.208.0.0/12": ["tag:rte-player"], "54.72.0.0/13":  ["tag:rte-player"],
    "54.216.0.0/13": ["tag:rte-player"], "63.32.0.0/14":  ["tag:rte-player"],
    // AWS eu-central-1 / Frankfurt (theplatform entitlement failover)
    "3.64.0.0/12":   ["tag:rte-player"], "3.120.0.0/13":  ["tag:rte-player"],
    "18.156.0.0/14": ["tag:rte-player"], "18.192.0.0/13": ["tag:rte-player"],
    "35.156.0.0/14": ["tag:rte-player"], "52.28.0.0/14":  ["tag:rte-player"],
    "52.56.0.0/14":  ["tag:rte-player"],
    // RTE origin + Youbora/NPAW QoS analytics
    "89.207.0.0/16": ["tag:rte-player"], "89.149.208.0/20":["tag:rte-player"],
  },
},
```

It is broad, but it is safely broad, because `autoApprovers` only approves routes your connector actually advertises for its domains, so nothing else gets pulled in. This list will go stale over time, so treat it as a starting point and lean on step 6. Also make sure your policy lets members reach the routes, which Tailscale's default `{"src": ["autogroup:member"], "dst": ["*"], "ip": ["*"]}` grant already does.

### 3. Turn the Ireland box into the connector

On the always on box:

```sh
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-tags=tag:rte-player --hostname=rte-gw
```

Then enable app connector mode on it, which is easiest in the admin console under App connectors by adding the machine to the `rte-player` connector. Advertise the static ranges from step 2 with `tailscale set --advertise-routes=...` (the full comma separated list of CIDRs), and enable IP forwarding if your distribution has not already. On OpenWrt I keep the routes in UCI so they survive a reboot.

### 4. Fix the tunnel MTU

The `tailscale0` MTU is 1280, and full sized video segments can be dropped if the segment size is not clamped. On both the gateway and any downstream router, drop in `/etc/nftables.d/20-tailscale-mss.nft`:

```
chain tailscale_mss_clamp {
    type filter hook forward priority mangle; policy accept;
    iifname "tailscale0" tcp flags syn tcp option maxseg size > 1240 tcp option maxseg size set 1240
    oifname "tailscale0" tcp flags syn tcp option maxseg size > 1240 tcp option maxseg size set 1240
}
```

On plain iptables the equivalent is `iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1240`, and the same with `-i tailscale0`.

### 5. The client

Install Tailscale on the phone or laptop, sign in, and turn on using subnet routes (a toggle in the mobile apps, or `tailscale up --accept-routes` on the command line). Leave the exit node off, since that is the whole point. RTE bound traffic now goes through Ireland and everything else takes the local link.

### 6. Find the routes you are missing

The address lists above are a snapshot and will drift. When a video will not play, do not guess, watch what the device asks for. If the client sits behind a small router running `dnsmasq`:

```sh
uci set dhcp.@dnsmasq[0].logqueries='1'; uci commit dhcp
/etc/init.d/dnsmasq restart
logread -f | grep <client-ip>          # then press play and watch
```

Resolve each hostname from the gateway, so you see the in country answer, and check every address against your enabled subnet routes while excluding `0.0.0.0/0`, or the exit node route will make everything look covered and hide the gap. Any streamer owned host that resolves to an uncovered address is a route you are missing. Add its range and you are done.
