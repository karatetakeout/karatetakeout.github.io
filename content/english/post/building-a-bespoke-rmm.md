+++
author = "Grant Houser"
title = "Building a Bespoke RMM"
date = "2026-02-23"
description = "How I saved $10k and Gained Full Network Visibility in 3 Days"
tags = ["cursor ai", "python", "powershell", "windows service, snmp"]
categories = ["ROI", "Cost Optimization", "RMM", "Proactive IT"]
+++

One of the primary concerns in a multi-unit hospitality environment is the "Visibility Gap." With nearly 300 endpoints across 30 physical locations in Middle and East Tennessee, the risk of a catastrophic hardware failure isn't just a technical problem—it’s a revenue problem. A fried server or a crashed POS terminal during a Friday night rush leads to immediate sales loss and operational chaos.

As a one-person IT department, I don’t have the luxury of manually checking the health of every device. I needed a way to manage upgrades, monitor hardware vitals, and handle patches proactively rather than reactively.

&nbsp;
## The RMM Challenge
This led me to research <b>Remote Monitoring and Management (RMM)</b> solutions.

> <b>What is an RMM?<b>
> An RMM is a platform designed to help IT professionals monitor and manage endpoints (servers, workstations, and mobile devices) from a centralized location. It provides real-time visibility into system health, automates routine maintenance, and allows for remote troubleshooting. The primary benefits are increased uptime, enhanced security through automated patching, and the ability to scale IT oversight without adding headcount.

The hurdle was the cost. Most RMM providers utilize a per-endpoint pricing model. After vetting several vendors, the quote to cover our footprint was approximately $10,000 per year. Even native solutions like Microsoft Intune carried a significant price tag and administrative overhead that made it a difficult sell for a lean organization.

&nbsp;
## The Pivot: Custom Build via AI-Assisted Development
I decided to leverage my existing infrastructure and modern AI tools to build a custom solution. Because I had already established an SD-WAN across our locations, I had a clear path for direct communication between endpoints and my corporate office without needing to provision an expensive cloud intermediary.

Preferring functional substance over a flashy interface, I opted for a Command Line Interface (CLI) approach. By skipping the UI/UX development phase, I could focus entirely on the logic and reliability of the system.

&nbsp;
## The Build Process
Using Cursor as my primary development environment, I was able to move from concept to deployment in roughly 2.5 days.

1. **The Client:** I developed a PowerShell script that captures CPU, RAM, and Disk usage data. It includes an install script that registers a Windows Scheduler event to poll and report data every 10 minutes.

2. **The Receiver:** A Python script, registered as a Windows service at the corporate office, acts as the listener to ingest and store incoming diagnostics.

3. **The Dashboard:** A CLI-driven program that allows me to:
    * Set thresholds for alerts (e.g., disk usage >90%).
    * Filter devices by type, name, or physical location.
    * Monitor peripherals like printers and network switches via SNMP polling.
    * View, start, or stop services on any remote endpoint.
    * Execute SSH commands directly from the management console.

&nbsp;
# The Result: $2 vs. $10,000
The result is a fully functional, company-owned RMM that runs on our existing infrastructure with zero SaaS fees.

The efficiency of using a tool like Cursor cannot be overstated. While I am a proficient Python scripter, a project of this scope would traditionally have taken weeks of dedicated coding. By leveraging AI-assisted development, I spent about $2 in subscription time and three days of work to save the company $10,000 annually.

More importantly, I’ve moved from a "break-fix" mentality to a proactive one. We still have features to add—patch management and MDM integration are next on the roadmap—but for now, I can finally see the health of our network in real-time. That visibility is the difference between a minor maintenance task and a catastrophic loss of sales.