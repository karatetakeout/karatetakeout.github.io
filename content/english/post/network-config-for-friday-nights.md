+++
author = "Grant Houser"
title = "Resilient Networking: Building for the Friday Night Rush"
date = "2026-02-08"
description = "Network configurations to help make operations run smoother"
tags = ["network admin", "vlan"]
categories = ["networking"]
+++

Configuring a network isn't just an exercise in security postures or connectivity. In the restaurant industry, your network is an operational tool. Its true litmus test isn't a Tuesday morning audit; it’s a slammed Friday night. If the network can’t facilitate a smooth service under pressure, it’s not doing its job.

When I took over the Network Administrator role at Aubrey’s, the landscape was fragmented. We had multiple third-party vendors "owning" different systems across our locations. Some buildings ran on two separate networks; others had one network but there was not always any clear ownership. It was a slow process, but we’ve successfully consolidated. Today, we own our networks—one per restaurant—giving us the control necessary to scale.

&nbsp;
## The Problem: Static Specs in a Dynamic Environment
This project was born out of necessity when we transitioned to a new POS system, InfoGenesis. Our previous provider was an all-in-one shop that handled everything from hardware to networking. However Agilysys, the parent company for InfoGenesis, focuses on the software, leaving the infrastructure to us.

Under a tight rollout deadline, our initial configurations were functional but basic. We focused on isolation and connectivity. It checked the boxes, but it didn't account for real-world hardware failures.

The main pain points were our credit card PIN pads and kitchen printers:

* **PIN Pads:** We used MAC reservations to assign unique IP addresses. If a PIN pad died, a manager couldn't just swap it out. The new hardware wouldn't have the correct IP, and the POS wouldn't "see" it.

* **Kitchen Printers:** These were manually assigned static IPs. If a printer failed during a rush, someone (usually me) had to manually configure a replacement or have a pre-configured backup on the shelf. With up to four printers per site and locations hours away, this was an expensive, inefficient insurance policy.

&nbsp;
## The Solution: Designing for "Plug-and-Play"
To solve this, I redesigned the configuration to prioritize field-level "plug-and-play" capability.

I moved away from static assignments and implemented a one-VLAN-per-device strategy using /30 CIDR masks. By creating these tiny, isolated networks, each one is only capable of assigning a single IP address.

Now, we set the devices to DHCP.

* **The Result:** If a kitchen printer fails, a manager can grab any backup printer, plug it in, and the network automatically assigns it the correct IP for that specific station.

* **The Benefit:** We no longer need a dedicated backup for every station. One "cold" backup can replace any printer in the building with zero configuration. It saves thousands in hardware costs and eliminates emergency support calls.

&nbsp;
Security and Performance
From a security standpoint, we haven't traded protection for convenience. We’ve implemented zone management, grouping these individual VLANs into a single POS Zone. This allows us to apply consistent ACL rules across the entire environment.

My primary concern with this move was latency. Because every request to a printer or PIN pad now has to hit the router instead of staying at the switch level, there was a risk of slowing down the transaction flow. To mitigate this, we’ve started deploying Layer 3 switches at our newer locations to handle that routing locally. So far, the performance has been seamless.

&nbsp;
## Looking Ahead
This reconfiguration also gave us the opportunity to migrate from Class C to Class A IP addressing. This move normalizes our internal addresses, which is a critical prerequisite as I begin building out our SD-WAN but, that’s a topic for another post.

For now, the win is simple: our managers can fix hardware issues in seconds, and I can stay focused on high-level infrastructure instead of emergency printer swaps.