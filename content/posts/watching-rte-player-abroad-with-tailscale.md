+++
categories = ["networking"]
date = "2026-07-01T00:00:00+01:00"
description = "A step by step guide to watching RTE Player from abroad with a Tailscale app connector, split tunnelling only the RTE traffic through Ireland."
keywords = ["tailscale", "app connector", "rte", "rte player", "geo-unblock", "openwrt", "wireguard", "subnet router", "streaming", "dns"]
title = "Watching RTE Player abroad with a Tailscale app connector"
+++

RTE Player, like most Irish broadcasters, is geo locked to Ireland. When I am abroad I still want to watch it, but I have no interest in routing my entire phone through a VPN to get at it. My banking app, my email and everything else can stay on the local connection. I only want the traffic that talks to RTE to look like it is coming from home.

The tidy way to do this is a [Tailscale](https://tailscale.com/) [app connector](https://tailscale.com/kb/1281/app-connectors). I run an always on box at home in Ireland that acts as both a subnet router and an app connector. It advertises routes for RTE's CDN ranges, my travel router and phone accept those routes, and only RTE bound traffic hairpins back through Ireland. Everything else takes the local link.

The guide below is RTE specific, but the shape is the same for any geo locked streamer. Only the domains and the CDN ranges change, and the last step tells you how to find yours.

## What you need

Three things:

- A Tailscale account. The free tier is fine.
- An always on Linux box in the country you want to appear from. Mine is an OpenWrt router, but a Raspberry Pi or a spare mini PC on a residential line works just as well. Residential matters, because streamers flag datacentre addresses.
- The client device running Tailscale.

## 1. Declare a tag and an app connector

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

One domain is easy to miss and worth calling out. RTE's video entitlement, the request that decides whether you are allowed to watch, is served from `link.eu.theplatform.com`, on `theplatform.com` rather than `theplatform.eu`. If you only watch the `.eu` name, the video segments download fine but the entitlement request leaks out over your local connection, arrives from the wrong country and gets refused. The app loads and the video never plays. Include both.

## 2. Auto approve the CDN ranges

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

## 3. Turn the Ireland box into the connector

On the always on box:

```sh
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-tags=tag:rte-player --hostname=rte-gw
```

Then enable app connector mode on it, which is easiest in the admin console under App connectors by adding the machine to the `rte-player` connector. Advertise the static ranges from step 2 with `tailscale set --advertise-routes=...` (the full comma separated list of CIDRs), and enable IP forwarding if your distribution has not already. On OpenWrt I keep the routes in UCI so they survive a reboot.

## 4. Fix the tunnel MTU

The `tailscale0` MTU is 1280, and full sized video segments can be dropped if the segment size is not clamped. On both the gateway and any downstream router, drop in `/etc/nftables.d/20-tailscale-mss.nft`:

```
chain tailscale_mss_clamp {
    type filter hook forward priority mangle; policy accept;
    iifname "tailscale0" tcp flags syn tcp option maxseg size > 1240 tcp option maxseg size set 1240
    oifname "tailscale0" tcp flags syn tcp option maxseg size > 1240 tcp option maxseg size set 1240
}
```

On plain iptables the equivalent is `iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1240`, and the same with `-i tailscale0`.

## 5. The client

Install Tailscale on the phone or laptop, sign in, and turn on using subnet routes (a toggle in the mobile apps, or `tailscale up --accept-routes` on the command line). Leave the exit node off, since that is the whole point. RTE bound traffic now goes through Ireland and everything else takes the local link.

## 6. Find the routes you are missing

The address lists above are a snapshot and will drift, so when a video will not play, do not guess, watch what the device asks for. If the client sits behind a small router running `dnsmasq`:

```sh
uci set dhcp.@dnsmasq[0].logqueries='1'; uci commit dhcp
/etc/init.d/dnsmasq restart
logread -f | grep <client-ip>          # then press play and watch
```

Resolve each hostname from the gateway, so you see the in country answer, and check every address against your enabled subnet routes while excluding `0.0.0.0/0`. If you leave the exit node route in the comparison it makes everything look covered and hides the gap. Any streamer owned host that resolves to an uncovered address is a route you are missing. Add its range and you are done.

Two things worth remembering. The host that has to appear in country is the entitlement or licence request, not the video itself, since the segments can stream from anywhere once you are allowed to watch. And if a full exit node plays the video but split routing does not, that is always a coverage gap and nothing more, so go and find the missing host.
