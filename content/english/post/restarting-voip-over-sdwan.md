+++
author = "Grant Houser"
title = "Restarting VoIP Over SD-WAN"
date = "2026-05-27"
description = "Giving managers the Power to Fix Their Own Phones"
tags = ["python", "powershell", "flask", "voip", "unifi"]
categories = ["automation", "networking"]
+++

One of the realities of running IT for a multi-unit restaurant group as a one-person department is that not every problem can wait for me. A phone system that goes down at 11am on a Saturday is a real operational problem — the host stand can't take reservations, managers can't reach vendors, and the general chaos of a busy service gets worse. The fix is usually a two-minute task. The problem is that the two minutes it takes me to stop what I'm doing and get involved is rarely when I'm available.

UniFi Talk has been a solid VoIP solution for us, but it has a quirk: occasionally the Talk service gets into a bad state and needs to be restarted to get the phones registering correctly again. It's not a crash, it's not a hardware failure — it's just a service that needs a kick. The fix is simple, but it requires admin access to the UDM Pro, which I'm not handing to store managers.

I needed a way to delegate just this one action, without delegating anything else.

---

## The Constraint: Delegate the Action, Not the Access

The core problem with most obvious solutions is that they conflate action with access. Giving a manager a UniFi login solves the immediate problem but opens up the entire network management interface. Sharing credentials is a security and audit nightmare. Writing the credentials into a script file on a store computer creates a different kind of exposure.

What I actually wanted was something closer to a locked key box — a mechanism where a manager can perform one specific action, in one specific place, without any visibility into how it works or what else could theoretically be done.

> **The Design Principle**
> The credential and the execution environment should never be accessible to the person triggering the action. The manager presses a button. That's it.

---

## The Architecture: A Corporate Jump Proxy

The solution I landed on uses the SD-WAN infrastructure I've been building out as a backbone. Rather than exposing the UDM Pro to any kind of direct management from the store, all actions flow through a lightweight Flask service running on the corporate Windows Server.

```
Manager double-clicks desktop shortcut
        ↓
PowerShell script on store office PC
        ↓  (HTTPS over SD-WAN)
Flask service on corporate server
        ↓  (SSH with key-based auth)
UDM Pro at store
        ↓
Talk service restarts
```

The SSH key never leaves the corporate server. The manager never sees a URL, a credential, or a command. The PowerShell script on the store PC contains only a store-specific secret token — and that token is only capable of triggering a Talk restart. You couldn't use it to touch a VLAN, reboot a switch, or do anything else on the network.

---

## Per-Store Secrets: Containing the Blast Radius

As I started building this out for our first store, I thought ahead to the eventual rollout across all of our locations. A single shared secret across all stores would mean that if one store's script was ever copied or compromised, it could theoretically restart phones at any location. That felt sloppy.

Instead, each store gets its own unique secret. The Flask service validates incoming requests against the secret for that specific store endpoint. A secret from one location sent to another location's endpoint gets a 403. The credential is scoped to exactly one store and one action.

When I add more stores, it's a three-step process: add an entry to the store registry, copy the SSH public key to that location's UDM Pro, and generate a new secret. The store PC gets its own version of the script with its own secret baked in.

---

## The Manager Experience

The manager's experience is deliberately simple:

1. They sit down at the office PC and double-click an icon on the desktop labeled "Restart Phones"
2. A dialog box appears warning them that any calls in progress will be dropped and asking them to confirm
3. A "please wait" screen appears while the request is in flight
4. A success or failure message tells them what happened

There's no login. There's no terminal window. There's no way to accidentally do something wrong. If something goes wrong on the backend, they get a message that says "contact IT" — which is the right outcome anyway since at that point it's beyond a simple service restart.

---

## Keeping It On the SD-WAN

One option I considered early on was using Power Automate as the trigger mechanism. The office PCs are already Entra ID joined, so the button experience would have been clean. I ultimately decided against it for two reasons.

First, Power Automate calling a private IP at the corporate office requires either a static WAN address or an on-premises data gateway punching a hole outward to Microsoft's cloud. Neither felt right for what is fundamentally an internal operation.

Second, and more importantly, everything about this problem exists entirely within our own infrastructure. The SD-WAN connects the stores to corporate. The corporate server runs the service. The UDM Pro sits on the store's LAN. Routing that control flow out through a cloud service and back in adds a dependency that doesn't need to exist. If the internet is down at corporate, the phones still need to be restartable.

Keeping it on the SD-WAN means the only failure mode is our own infrastructure — which is something I can monitor and control.

---

## The Stack

For anyone interested in the technical details:

- **Flask** (Python 3.13) running as a Windows Service via NSSM on the corporate server
- **Self-signed TLS certificate** on port 8443 — internal only, no need for a public CA
- **ED25519 SSH key pair** for UDM Pro authentication — no passwords involved anywhere
- **PowerShell 5.1** on the store PC with Windows Forms dialogs for the manager UI
- **Per-store secret tokens** generated with Python's `secrets` module and stored as NSSM environment variables — never on disk in plain text

The Flask service logs every restart attempt with a timestamp, store name, caller IP, and success or failure detail. I have a full audit trail of every time a manager has used it.

---

## The Broader Pattern

What I like about this approach is that it scales cleanly. The corporate server is already becoming a control plane for the SD-WAN — the RMM receiver lives there, and now the Talk restart service does too. As I onboard more stores, I'm adding entries to a config file rather than building new infrastructure.

More importantly, it establishes a pattern for delegating other specific actions in the future without delegating access. If there's another "just restart this thing" problem at a store — and in this industry there always is — I have a framework to wrap it safely.

The goal was never to give managers less capability. It was to give them exactly the capability they need, packaged in a way that's safe, auditable, and impossible to misuse.
