# SRX Advanced Security Policies

Hands-on Juniper SRX security policy labs focused on application-based traffic matching, policy actions, and local session-log verification.

## Lab environment

| Item | Value |
|---|---|
| Firewall | Juniper SRX – HQ |
| Source zone | `internal` |
| Destination zone | `branch` |
| Test host | Ubuntu |
| Test host address | `192.168.30.2/32` |
| Test destination | `8.8.8.8` |

## Current lab

**Application-Based Security Policy Matching and Logging**

- Match a specific source host using the Global Address Book.
- Deny DNS over UDP and TCP using Junos predefined applications.
- Temporarily test ICMP matching with `junos-ping`.
- Verify denied traffic through local `RT_FLOW_SESSION_DENY` log entries.

Documentation: [Advanced Security Policy Matching](docs/advanced-security-policy-matching.md)

## Repository structure

```text
SRX-Advanced-Security-Policies/
├── README.md
└── docs/
    ├── advanced-security-policy-matching.md
    └── images/
        └── .gitkeep
```

Topology and lab screenshots will be added under `docs/images/`.