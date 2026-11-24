# SRX Advanced Security Policies

A practical Juniper SRX lab repository covering security policy configuration, application matching, traffic enforcement, and verification using firewall session logs.

This repository records the configuration and results of each experiment. The current lab focuses on controlling traffic from one Ubuntu endpoint by matching Junos predefined applications.

## Project objectives

- Identify a specific endpoint with a Global Address Book object.
- Match traffic using Junos predefined applications.
- Deny DNS over UDP and TCP independently.
- Temporarily test ICMP Echo Request matching.
- Verify policy enforcement using local `RT_FLOW_SESSION_DENY` logs on the SRX.
- Preserve the actual configuration, test commands, observed output, and topology for repeatability.

## Lab topology

![SRX Advanced Policy Lab Topology](docs/images/SRX%20Advanced%20Policy%20Topology.JPG)

The test endpoint is **PC1 (Ubuntu)** in VLAN 30. Its traffic passes through `L3_SW` to the HQ SRX firewall (`HQ-Node0`).

| Component | Details |
|---|---|
| Test endpoint | PC1 – Ubuntu |
| Test endpoint IP | `192.168.30.2/32` |
| Endpoint VLAN | VLAN 30 |
| VLAN 30 subnet | `192.168.30.0/29` |
| VLAN 30 gateway | `192.168.30.1` |
| Switching device | `L3_SW` |
| Firewall | `HQ-Node0` – Juniper SRX |
| SRX security zones | `internal` → `branch` |
| SRX-to-switch transit network | `10.10.10.0/30` |
| SRX transit IP | `10.10.10.1` |
| L3 switch transit IP | `10.10.10.2` |
| Test destination | `8.8.8.8` |

VLANs 10 and 20 are also shown in the wider topology, but this experiment uses the Ubuntu endpoint in VLAN 30.

**Traffic path under test:** PC1 (Ubuntu) → L3_SW → HQ-Node0 (SRX) → destination-side path.

## Experiment 1 – Application-based security policy matching

### 1.1 Global Address Book object

The Ubuntu endpoint was defined in the Global Address Book:

```bash
set security address-book global address ubuntu-user-ip 192.168.30.2/32
```

### 1.2 Security policy

A dedicated policy was created from zone `internal` to zone `branch`:

- **Policy name:** `internal-branch_ubuntu-user-test`
- **Source address:** `ubuntu-user-ip`
- **Destination address:** `any`
- **Action:** `deny`
- **Logging:** `session-init` and `session-close`

The application matches were added during the experiment:

```bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match source-address ubuntu-user-ip
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match destination-address any
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match application junos-dns-udp
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test then deny
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test then log session-init
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test then log session-close
```

DNS over TCP was then added to the same policy:

```bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match application junos-dns-tcp
```

For the ICMP test, `junos-ping` was added temporarily and then rolled back:

```bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match application junos-ping
```

**Final state after the experiment:** DNS over UDP and TCP remained in the deny policy. The temporary ICMP match was rolled back and is not part of the final DNS-filtering policy.

## Experiment 2 – DNS over UDP

The first stage matched `junos-dns-udp` and denied the matching DNS traffic.

### Ubuntu test commands

```bash
ping 8.8.8.8
nslookup 8.8.8.8
curl -I https://google.com
```

### Observed Ubuntu results

**ICMP:** The ping test received replies from `8.8.8.8`, showing that ICMP was not blocked by the DNS-only policy during this stage.

**DNS lookup:** The `nslookup 8.8.8.8` command timed out through the local Ubuntu stub resolver:

```text
;; communications error to 127.0.0.53#53: timed out
;; communications error to 127.0.0.53#53: timed out
;; communications error to 127.0.0.53#53: timed out
;; no servers could be reached
```

**HTTPS request:** The initial curl request succeeded and returned an HTTP response:

```text
HTTP/2 301
location: https://www.google.com/
content-type: text/html; charset=UTF-8
server: gws
```

The initial successful curl result does not prove that DNS UDP was passing at that moment; a cached DNS answer may affect the result. The firewall log below is the direct evidence that DNS UDP requests to `8.8.8.8:53` were denied.

### SRX verification – UDP deny log

Command executed on the SRX:

```bash
run show log messages | match RT_FLOW_SESSION | match udp
```

Representative log entry captured from the SRX:

```text
Nov 23 19:51:42 HQ RT_FLOW: RT_FLOW_SESSION_DENY:
session denied 192.168.30.2/51154->8.8.8.8/53 0x0
junos-dns-udp 17(0) internal-branch_ubuntu-user-test
internal branch ... ge-0/0/0.0 No Denied by policy
```

**What this proves:**

- `RT_FLOW_SESSION_DENY`: the SRX denied the session.
- `192.168.30.2 → 8.8.8.8`: source and destination IP addresses.
- `8.8.8.8/53`: DNS destination port 53.
- `junos-dns-udp`: matched Junos application.
- `17(0)`: IP protocol 17 (UDP).
- `internal-branch_ubuntu-user-test`: policy that denied the traffic.
- `Denied by policy`: firewall denial reason.

The SRX produced multiple UDP deny entries during the test, not just the single representative entry shown above.

## Experiment 3 – DNS over TCP

The following application was added to the same deny policy:

```bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match application junos-dns-tcp
```

### Ubuntu test command

```bash
curl -I https://google.com
```

### Observed Ubuntu result

```text
curl: (6) Could not resolve host: google.com
```

The hostname could not be resolved, so curl could not proceed to establish the HTTPS connection.

### SRX verification – TCP deny log

Command executed on the SRX:

```bash
run show log messages | match RT_FLOW_SESSION | match tcp
```

Representative log entry captured from the SRX:

```text
Nov 23 19:56:46 HQ RT_FLOW: RT_FLOW_SESSION_DENY:
session denied 192.168.30.2/44748->8.8.8.8/53 0x0
junos-dns-tcp 6(0) internal-branch_ubuntu-user-test
internal branch ... ge-0/0/0.0 No Denied by policy
```

**What this proves:**

- `RT_FLOW_SESSION_DENY`: the SRX denied the session.
- `192.168.30.2 → 8.8.8.8:53`: DNS traffic from the Ubuntu endpoint to port 53.
- `junos-dns-tcp`: matched Junos DNS-over-TCP application.
- `6(0)`: IP protocol 6 (TCP).
- `internal-branch_ubuntu-user-test`: policy that denied the traffic.
- `Denied by policy`: firewall denial reason.

The SRX recorded repeated TCP deny entries during the test.

## Experiment 4 – Temporary ICMP blocking

A continuous ping test from Ubuntu to `8.8.8.8` was running. During the test, the following match was added to the deny policy:

```bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-test match application junos-ping
```

The ping output showed replies before the policy change, an interruption while the ICMP match was active, and replies again after the policy was rolled back.

### SRX verification – ICMP deny log

Command executed on the SRX:

```bash
run show log messages | match RT_FLOW_SESSION | match icmp
```

Representative log entry captured from the SRX:

```text
Nov 23 19:58:08 HQ RT_FLOW: RT_FLOW_SESSION_DENY:
session denied 192.168.30.2/12->8.8.8.8/11 0x0
icmp 1(8) internal-branch_ubuntu-user-test
internal branch ... ge-0/0/0.0 No Denied by policy
```

**What this proves:**

- `RT_FLOW_SESSION_DENY`: the SRX denied the ICMP session.
- `icmp 1(8)`: ICMP protocol 1, Echo Request type 8.
- `internal-branch_ubuntu-user-test`: policy that denied the traffic.
- `Denied by policy`: the firewall denied the matching Echo Request.

The ICMP match was rolled back after the experiment. The log entries confirm that ICMP Echo Requests were denied while that temporary match was active.

## Verification summary

| Traffic | Junos application | Protocol | Observed outcome | SRX log evidence |
|---|---|---|---|---|
| DNS over UDP | `junos-dns-udp` | UDP (17), destination port 53 | DNS requests denied | `RT_FLOW_SESSION_DENY` |
| DNS over TCP | `junos-dns-tcp` | TCP (6), destination port 53 | Domain resolution failed in the reported curl test | `RT_FLOW_SESSION_DENY` |
| ICMP Echo Request | Temporary `junos-ping` | ICMP (1), type 8 | Echo Requests denied while the match was active | `RT_FLOW_SESSION_DENY` |

## Local session-log verification commands

All verification was performed locally on the SRX using the `messages` log:

```bash
run show log messages | match RT_FLOW_SESSION | match udp
run show log messages | match RT_FLOW_SESSION | match tcp
run show log messages | match RT_FLOW_SESSION | match icmp
```

The important log fields are the source and destination, matched application or protocol, policy name, and the denial reason. The repeated `RT_FLOW_SESSION_DENY` entries show that the SRX processed and denied matching traffic from the Ubuntu endpoint.

## Conclusion

This experiment verified application-based policy matching and deny enforcement on the Juniper SRX:

1. A Global Address Book object identified the Ubuntu source host.
2. DNS over UDP was denied using `junos-dns-udp`.
3. DNS over TCP was denied using `junos-dns-tcp`.
4. ICMP Echo Requests were temporarily denied using `junos-ping`.
5. Local SRX `RT_FLOW_SESSION_DENY` entries confirmed which policy denied each tested application.

The current policy retains the DNS UDP and TCP matches. The ICMP match was temporary and was rolled back.

## Repository structure

```text
SRX-Advanced-Security-Policies/
├── README.md
└── docs/
    ├── advanced-security-policy-matching.md
    └── images/
        ├── .gitkeep
        └── SRX Advanced Policy Topology.JPG
```

## Detailed documentation

See [Application-Based Security Policy Matching and Logging](docs/advanced-security-policy-matching.md) for the full lab notes and configuration walkthrough.

## Future experiments

Additional topics will be added only after they have been configured, tested, and verified. Planned candidates include Address Sets, custom applications, policy ordering, policy scheduling, and further policy troubleshooting.
