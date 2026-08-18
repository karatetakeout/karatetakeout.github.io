---
title: "Clocking In Without the iPads"
date: 2026-08-18
description: "How a custom Windows kiosk app on our existing POS terminals let us skip iPads, MDM, and mounting costs for our Hot Schedules rollout"
categories: ["cost optimization", "restaurant tech"]
tags: ["wpf", "webview2", "azure-key-vault", "infogenesis", "hotschedules", "kiosk"]
---

We're rolling out Hot Schedules across all of our locations over the next year, and the standard plan for the web clock looked like most restaurant tech rollouts: buy some iPads for each store, enroll them in MDM, mount them somewhere near the POS, and pull power (and possibly new wiring) to make the mounts work. Multiply that by every location and it adds up before a single employee ever clocks in — hardware, recurring MDM subscriptions, and installation labor, all for a device that does one job.

## The iPad math

None of those costs are avoidable if the plan is a dedicated device. An iPad needs to be provisioned, enrolled, secured, and physically mounted somewhere an employee can reach it during a rush. At scale, across dozens of locations, that's real capital spend plus an ongoing subscription, before accounting for the electrician time some of our older buildings would need just to get power to the right spot on the wall.

## Rethinking the terminal

Every one of our locations already runs InfoGenesis POS terminals — Windows 10 IoT machines with touchscreens sitting right at the point of sale. InfoGenesis supports custom function buttons that can launch an external application from inside a POS session. That's the opening: if the Hot Schedules web clock could live behind a button on a screen that's already there, we don't need a second device at all.

## Building the clock app

I built a small Windows app (.NET 8, WPF) that wraps the Hot Schedules web clock in a WebView2 control — the same Chromium engine that powers Edge, just embedded directly in the app instead of a full browser window. A few things it had to handle:

- **Singleton behavior.** If the function button gets pressed while the app's already open, it brings the existing window to the front instead of stacking up a second instance.
- **Auto-login.** The app detects whether the clock is showing a login form and fills it in automatically, so staff hit the button and land straight on the punch screen.
- **No taskbar exposure.** POS terminals hide the Windows taskbar entirely — a real security requirement, not a cosmetic one. The app runs borderless and topmost, and blocks the usual escape hatches (Alt+Tab, the Windows key, the system menu) while it has focus, so it can't become a door back to the desktop.

> **What's WebView2, and why not just open a browser?**
> WebView2 lets a desktop app embed a real Chromium browser engine as a UI control, instead of shelling out to an actual browser window. That means no address bar, no tabs, no browser chrome a server could tap their way out through — just the clock-in page, full screen, and nothing else.

## Centralized credentials, not another password taped to a monitor

The clock login needed to live somewhere that wasn't hardcoded into the app or written down at each terminal. I put it in Azure Key Vault, with each terminal authenticating via a certificate rather than a stored secret. The app checks a local encrypted cache first (so a terminal keeps working through a brief internet hiccup mid-shift) and refreshes from the vault periodically. Rotating the password now means updating one secret in Key Vault — every terminal picks it up on its next refresh, with no redeploy and no visit to each location.

## The result

No iPads. No MDM subscription. No mounting hardware, and no electrician visits to run power to a new fixture. The app installs by copying one file to a folder and wiring up a function button — the same terminal that was already ringing in orders now handles the clock-in too.

## The real advantage: configurability

Working through this made something clearer to me than I expected going in. iPads (and locked-down mobile devices generally) are built to resist exactly this kind of customization — that's the point of them, for a lot of use cases. A Windows-based POS terminal is a full, general-purpose computer that happens to be running point-of-sale software. That's a liability if it's not locked down properly, but it's also the whole reason this project was possible at all: I could reach in, add a function, and integrate it directly into hardware we already owned. That flexibility is one of the more underrated advantages Windows terminals have over iPads and similar devices in this industry — and it's the difference between buying new hardware for every new tool and just building the tool.