# Software-Defined 4G LTE Lab

A complete, software-only 4G LTE network built and debugged from scratch — no SDR, no RF hardware, no SIM card. Every layer of the stack, from the radio interface down to the core network, runs as software on a single Ubuntu machine.

This repo documents the full build: the architecture, the working configuration, and — just as importantly — the real debugging process that got it from "nothing runs" to a UE successfully attaching, getting an IP, and passing traffic to the internet.

---

## Architecture

```
                    ┌──────────────────────────┐
                    │        Internet          │
                    └────────────┬─────────────┘
                                 │
                         ┌───────▼───────┐
                         │    Open5GS    │
                         │   4G EPC/Core │
                         │               │
                         │ MME  HSS      │
                         │ SGW  PGW      │
                         └───────┬───────┘
                                 │ S1 (S1-MME / S1-U)
                         ┌───────▼───────┐
                         │    srsRAN     │
                         │    eNodeB     │
                         └───────┬───────┘
                                 │
                              ZeroMQ
                          (virtual radio)
                                 │
                         ┌───────▼───────┐
                         │    srsRAN     │
                         │       UE      │
                         └───────────────┘
```

| Layer | Component | Role |
|---|---|---|
| Core network | **Open5GS** | MME, HSS, SGW, PGW — authentication, mobility management, session/bearer setup, IP allocation |
| Radio access | **srsRAN 4G** (eNodeB) | LTE air interface, scheduling, RRC |
| Terminal | **srsRAN 4G** (UE) | Simulated mobile device — cell search, attach, data |
| Virtual radio | **ZeroMQ** | Replaces RF entirely — I/Q samples exchanged over TCP instead of over the air |
| Analysis | **Wireshark / TShark** | S1AP, NAS-EPS, and GTP-U packet inspection |
| Host OS | Ubuntu 26.04.1 LTS | Everything built from source / native packages, no containers (yet) |

## Why build this

Real LTE/5G test equipment costs tens of thousands of dollars in SDR hardware and licensing. Running the entire signaling stack in software makes it possible to actually see — and break — every step of the attach procedure: RACH, RRC setup, NAS authentication, security mode, bearer establishment, and IP allocation, all on a laptop.

## Build phases

1. Environment prep (build deps, MongoDB, kernel/namespace checks)
2. Open5GS EPC — MME / HSS / SGW / PGW
3. Test subscriber provisioning (IMSI / K / OPc in the HSS)
4. srsRAN eNodeB build & configuration
5. ZeroMQ virtual RF link
6. srsRAN UE build & configuration
7. First attach — UE ↔ eNodeB ↔ core
8. User-plane traffic (ping / iperf through the GTP tunnel)
9. Wireshark / TShark analysis of the full signaling exchange
10. Idle mode & detach procedures
11. Deliberate failure injection (wrong keys, dropped associations, etc.)
12. Automation of test scenarios (Bash/Python)
13. Telemetry collection → feeding into an ML-based anomaly detection pipeline (in progress)

## Result: a real, working attach

With the core, eNodeB, and UE all running, the UE performs a full LTE attach:

```
Found Cell:  Mode=FDD, PCI=1, PRB=50, Ports=1, CP=Normal
Found PLMN:  Id=00101, TAC=7
Random Access Transmission: seq=17, ra-rnti=0x2
RRC Connected
Random Access Complete.  c-rnti=0x46
Network attach successful. IP: 10.45.0.3
```

Captured on the S1 interface between the eNodeB and Open5GS's MME, the complete EMM/ESM signaling for the attach — Identity, Authentication, Security Mode, ESM Information, bearer setup, all of it — reconstructed with Wireshark's Flow Graph:

![Full core-side signaling flow graph](./images/full_core_side_signaling.png)

**Attach Request → Attach Complete: ~0.59 seconds.** Full message-by-message breakdown, the NAS-only view, and how to reproduce the capture yourself are in [`CAPTURES.md`](./CAPTURES.md).

End-to-end user-plane traffic (UE → GTP tunnel → Open5GS UPF → real internet) was confirmed with `ping` and verified at the packet level on the UPF's `ogstun` interface with `tcpdump`.

See [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md) for the real issues hit along the way and how each was diagnosed and fixed — that's most of the actual engineering work.

## What's next

- Deliberate failure injection (wrong K/OPc, forced SCTP resets, RF link drops) to generate labeled failure telemetry
- Automating attach/detach cycles and traffic generation
- Feeding collected telemetry into a scikit-learn based anomaly/failure detection pipeline

## Disclaimer

This is a personal lab environment for learning the LTE protocol stack. It uses no real spectrum, no real subscriber data, and no radio transmission of any kind — all "radio" samples are exchanged locally over ZeroMQ/TCP. Configuration files in this repo have had subscriber secrets (K, OPc) redacted; only the well-known Open5GS test IMSI is shown.
