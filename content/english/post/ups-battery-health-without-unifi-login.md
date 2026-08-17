---
title: "UPS Battery Health, Without a UniFi Login"
date: 2026-08-17
description: "Giving Store Managers Visibility Into Their Backup Power"
categories: ["automation", "networking"]
tags: ["powershell", "unifi", "sharepoint", "entra", "sd-wan", "nut"]
---

Our newest location — still mid-build-out — is the first to get a pair of UniFi UPS units, a UPS-2U Pro and a UPS Tower, protecting the POS terminals and network gear from the kind of short outage that would otherwise mean a dead register mid-rush. That part was easy. The harder question came right after: how does a store manager know when a battery is actually failing, before it fails during service?

The honest answer, out of the box, is that they don't. That information lives in UniFi's own management portals — the Network app, Site Manager — none of which I have any interest in putting in front of a restaurant manager. It's one store today, which I could still check myself without much trouble. But we run close to 30 locations, and if this UPS rollout goes the way most good ideas around here do, it won't stay at one for long. I'd rather build the delegation pattern now, with exactly one console to get right, than retrofit it onto two dozen later. I don't want to train dozens of people on navigating a network controller just so they can glance at a battery percentage, and I definitely don't want to be the one manually checking every console myself every week.

I wanted the information delegated. Not the access.

---

## The Design Principle

This is the same shape of problem I ran into with [restarting VoIP phones over SD-WAN](https://granthouser.info/post/restarting-voip-over-sdwan/) a few months back — a manager needs to *know something* or *do something* on the network, without ever touching the network itself.

> **The Design Principle** The person who needs the information should never need the credentials that produce it. Give them the answer, not the login.

For the phones, that meant a locked-down action with no visibility into how it worked. For batteries, it meant the reverse: full visibility into a status, with zero access to the system generating it.

---

## The Long Way Around

I'll admit the path here wasn't a straight line, and I think the detours are worth writing down, because they're a decent tour of where UniFi's API story actually stands right now.

My first instinct was the obvious one: UniFi has an official Network Integration API, authenticated with a key generated right in the console. Straightforward, until I tried to reach it from outside the local network — our automation needed to run from a jumpbox, not from inside each store's LAN, so a Cloud Connector Proxy call through `api.ui.com` seemed like the right move.

That's where I ran into UniFi Fabrics. We'd grouped our 30 locations under a single Fabric for centralized identity and access management, which is genuinely useful — except it turns out Fabric-attached sites use a completely separate permission model from the classic Site Manager API. My existing API key authenticated fine and still got a flat `403` on every proxied call. A Fabric-specific API key solved the *authentication* problem, but the UPS telemetry itself — battery charge, load, runtime — simply isn't exposed in the official API's device statistics yet. New device category, API still catching up.

The actual answer was much older technology than any of that. Both UPS models ship with a built-in NUT (Network UPS Tools) server — a decades-old, IETF-documented protocol built for exactly this purpose. No API key, no cloud proxy, no Fabric permissions. Just a TCP connection to the UPS's own IP on port 3493, and it hands back everything: `battery.charge`, `battery.runtime`, `ups.load`, and a standardized `ups.status` code that tells you plainly when a unit is on battery power or flagging itself for replacement.

Sometimes the newest official integration isn't the right tool. The UPS has been answering this question the whole time; I just had to ask it directly.

---

## The Architecture

```fallback
UniFi UPS units (NUT server, port 3493)
        ↓  (SD-WAN, POS zone → UPS device IPs only)
PowerShell script on the corporate jumpbox (Task Scheduler, hourly)
        ↓  (Microsoft Graph, app-only auth via Entra)
SharePoint list, cleared and rewritten every run
        ↓
Grouped, color-coded view — no UniFi login required
```

The jumpbox is the same one running the VoIP restart service, on the same SD-WAN backbone. That's not a coincidence — once a corporate box has a properly firewalled path into store-level infrastructure, it becomes the obvious place to run anything that needs to talk to it without exposing that infrastructure directly. I could have run this same PowerShell script from a machine sitting in the store's own office. I didn't, for two reasons. First, scale: a script living on the jumpbox is one thing to maintain as this rolls out to more locations, instead of one thing per store. Second, and just as important: the jumpbox isn't on-premise, and it's locked down by firewall rules the same store staff have no access to. A manager can't poke at it, misconfigure it, or even reach it — the same instinct behind never handing them a UniFi login in the first place.

Getting a path open for this specific traffic still took a real firewall change, not just reusing what was already there. Our zone-based firewall segments the corporate network into zones, and the jumpbox sits in a POS zone with no default reason to reach anything at a store. I ended up not needing to open the store's UDM Pro at all — the rule only needed to allow traffic from the jumpbox's zone directly to the two UPS units' own IPs, nothing broader. I also set DHCP reservations on both UPS units so their addresses can't drift out from under the firewall rule (or the script's config) down the road. As anyone who's fought UniFi's zone firewall before knows, getting ICMP and the actual application port both passing cleanly still took a couple of rounds of troubleshooting the rule's return-traffic settings before it worked end to end.

Every hour, the script queries both UPS units via NUT, derives a health status from the standardized status flags (`RB`/`LB` for a battery that needs replacing, `OB` for a unit currently running on battery power), and pushes the result to SharePoint.

---

## Least Privilege, All the Way to SharePoint

The SharePoint side got the same "give exactly what's needed, nothing more" treatment as the UniFi side. Rather than a broad `Sites.ReadWrite.All` grant, the Entra app registration uses `Sites.Selected` — a permission type that, on its own, grants access to nothing. It has to be explicitly authorized against one specific site via a separate Graph call. The app that writes battery data can touch exactly one SharePoint site in our tenant and nothing else, which is the same instinct behind scoping the VoIP restart secrets per store: contain the blast radius of anything going wrong.

The list itself is deliberately dumb. It doesn't keep history — every run clears it and reinserts fresh rows, because a manager checking battery health cares about *right now*, not a log of every reading since installation. That simplicity turned out to be more reliable than the alternative, too: an earlier version tried to update existing rows in place, which meant depending on a filtered lookup against an indexed column behaving correctly on every run. Wipe-and-reinsert removes an entire category of thing that can quietly go wrong.

---

## The Manager Experience

Right now there's exactly one location's section to look at, but the view is already grouped by store — so the manager-facing experience won't change at all as more locations get UPS units. It's a filtered, grouped SharePoint list — nothing else:

1. Open a bookmarked link. No login flow beyond however the org already handles SharePoint.
2. See their location's section, already expanded or one click away.
3. See a battery percentage next to a colored status pill — green, yellow, or red.
4. Know, without asking IT, whether anything needs attention.

There's no portal to explain, no jargon like "runtime minutes" or "line-interactive" to define. Green means fine. Red means call IT. That's the entire interface.

---

## The Stack

For anyone building something similar:

- **PowerShell 5.1** on the corporate jumpbox, invoking `upsc.exe` (NUT's client tool) against each UPS
- **Windows Task Scheduler**, hourly, no always-on process required
- **UniFi's zone-based firewall**, with an explicit allow rule from the jumpbox's zone directly to each UPS unit's IP — never the store gateway
- **DHCP reservations** on each UPS unit, so their IPs stay fixed and the firewall rule (and the script) don't quietly break later
- **Microsoft Entra ID** app registration using `Sites.Selected` — scoped to exactly one SharePoint site
- **Microsoft Graph**, client-credentials flow, no interactive user in the loop anywhere
- **SharePoint list** with column formatting (JSON-based conditional styling) doing the green/yellow/red coloring natively, no custom UI

Nothing here needed a database, a hosted web app, or a dashboard framework. The pieces already existed — UniFi's hardware-level protocol, our own SD-WAN, and the Microsoft 365 tenant every manager already lives in.

---

## The Broader Pattern

This is the second time the SD-WAN and jumpbox combination has quietly done the hard part of a project that looks, on the surface, like it's about something else entirely. The VoIP restart service proved the backbone could safely delegate an *action*. This one proves the same backbone can safely delegate *visibility*. Neither required opening the network up — both required the opposite: finding the narrowest possible slice of capability and routing it through infrastructure that was already trusted and already there.

The pattern generalizes past batteries and phones. Any time the question is "how do I get one specific piece of store-level information or control into the hands of someone who shouldn't have broader access," the answer increasingly starts the same way: what does the jumpbox already have a path to, and what's the smallest possible key I can hand someone to unlock just this one door.