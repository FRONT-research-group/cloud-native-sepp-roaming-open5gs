# Open5GS SEPP Roaming Single-VM Lab

A conceptual and reproducible 5G SA roaming lab using **Open5GS**, **Docker Compose**, **SEPP-to-SEPP communication**, and **PacketRusher**.

---

## 1. What This Repository Demonstrates

This lab demonstrates a simplified 5G roaming scenario with two independent PLMNs running on a single Linux VM:

| Domain | Role |
|---|---|
| Home PLMN | Owns the subscriber and performs authentication |
| Visited PLMN | Provides access, mobility handling, session control, policy, and local user-plane breakout |
| SEPP NF| Provides the inter-PLMN signaling path between the two cores |
| PacketRusher | Simulates the UE and gNB from the visited side |

The roaming model implemented here is **Local Breakout (LBO)**:

- authentication is handled by the **Home PLMN**,
- PDU session control is handled by the **Visited SMF**,
- user-plane traffic exits through the **Visited UPF**.

This makes the lab useful for understanding the difference between:

```text
Who authenticates the subscriber?   -> Home PLMN
Where does the user plane break out? -> Visited PLMN
How does inter-PLMN signaling cross? -> SEPP-to-SEPP path
```

---

## 2. Why This Lab Exists

Most single-host 5G core labs place all NFs on the same network. That is useful for a basic 5G SA setup, but it hides the key idea behind roaming: **the Home PLMN and Visited PLMN are different administrative domains**.

This repository keeps that separation visible:

```text
Home PLMN Docker network      -> o5gs_h_net
Visited PLMN Docker network   -> o5gs_v_net
SEPP interconnect network     -> o5gs_interconnect_net
```

Only the SEPPs are connected to the interconnect network.

That design makes the lab easier to reason about:

- local NFs talk inside their own PLMN,
- inter-PLMN signaling goes through SEPP,
- user-plane traffic is locally broken out in the visited side.

---

## 3. PLMN Model

| Role | MCC | MNC | Label |
|---|---:|---:|---|
| Home PLMN | `001` | `01` | `mnc001.mcc001` |
| Visited PLMN | `999` | `70` | `mnc070.mcc999` |

The UE belongs to the Home PLMN:

```text
UE HPLMN = 001/01
```

The UE attaches through the Visited PLMN:

```text
Serving PLMN = 999/70
```
This is crucial, so that the visited AMF treats the UE as a roaming subscriber due to UE's identity refers to the Home PLMN.

---

## 4. Network Model

| Docker network | Subnet | Purpose |
|---|---|---|
| `o5gs_h_net` | `10.10.10.0/24` | Home PLMN internal SBI network |
| `o5gs_v_net` | `10.10.20.0/24` | Visited PLMN internal SBI network |
| `o5gs_interconnect_net` | `10.10.30.0/24` | SEPP-to-SEPP interconnect |

The key design rule is simple:

> Only `h-sepp` and `v-sepp` are connected to `o5gs_interconnect_net`.

All other NFs remain isolated inside their own PLMN network.

In both Docker Compose projects, the interconnect is referenced as an external network:

```yaml
networks:
  o5gs_interconnect_net:
    external: true
```

This is the first important Compose detail. The interconnect network is not owned by either the Home or Visited Compose project. It is a shared Docker bridge used only for SEPP-to-SEPP communication.

---

## 5. Conceptual Architecture

![alt text](blob:https://markdownviewer.pages.dev/9faf4c32-45e6-4b68-909e-f084d60a7948)

---

## 6. SEPP as the Roaming Boundary

The SEPP is the most important NF in this lab.

In this repository, each SEPP has two logical identities:

| SEPP role | Meaning |
|---|---|
| Local SBI identity | Used by local NFs inside the same PLMN |
| Peer/N32 identity | Used by the remote SEPP over the interconnect network |

For example, the Home SEPP uses:

| Purpose | FQDN | IP |
|---|---|---|
| Local SBI | `sepp.5gc.mnc001.mcc001.3gppnetwork.org` | `10.10.10.13` |
| Peer/N32 | `sepp-n32.5gc.mnc001.mcc001.3gppnetwork.org` | `10.10.30.2` |

The Visited SEPP uses:

| Purpose | FQDN | IP |
|---|---|---|
| Local SBI | `sepp.5gc.mnc070.mcc999.3gppnetwork.org` | `10.10.20.13` |
| Peer/N32 | `sepp-n32.5gc.mnc070.mcc999.3gppnetwork.org` | `10.10.30.3` |

This split is important because it prevents peer SEPP traffic from landing on the local SBI side.

---

## 7. N32-c and N32-f in This Lab

In a conceptual 5G roaming model, N32 can be understood as two logical parts:

| Logical part | Role |
|---|---|
| `N32-c` | SEPP capability exchange and control/negotiation |
| `N32-f` | Forwarding path for inter-PLMN signaling |

This repository does not try to implement a production-grade separated N32-c/N32-f deployment. Instead, Open5GS is configured with one `n32` peer endpoint per SEPP for roaming communication scenario.

The important learning objective is not to reproduce a full operator IPX/SCP/SEPP deployment. The objective is to show how such an integration works and roaming signaling crosses the PLMN boundary through SEPP.

---

## 8. Roaming Authentication Flow

![alt text](blob:https://markdownviewer.pages.dev/517db73d-d924-4303-98b5-976a009d3b38)

The Home PLMN owns the subscriber. Therefore:

```text
h-AUSF -> h-UDM -> h-UDR
```

is the subscriber authentication path.

The Visited PLMN does not authenticate the UE as a local subscriber. It routes the authentication request to the Home PLMN through SEPP.

---

## 9. Local Breakout Session Flow

After registration and authentication succeed, the UE requests a PDU session.

In this lab, the PDU session is handled locally in the Visited PLMN:

```text
PacketRusher UE -> v-AMF -> v-SMF -> v-PCF -> v-UPF
```

The key point is:

```text
Home PLMN authenticates the subscriber.
Visited PLMN anchors the user plane.
```

The UE PDU session pool is:

```text
10.45.0.0/16
```

This is not a Docker subnet. It is the UE address pool assigned by the SMF/UPF side.

---

## 10. How the Configuration Maps to the Theory
This repository is organized so each theoretical concept maps to a concrete file.

| Concept | Configuration file | What to look for |
|---|---|---|
| Home PLMN network advertising | `home_network/configs/nrf.yaml` | `serving.plmn_id: 001/01` |
| Visited PLMN network advertising | `visited_network/configs/nrf.yaml` | `serving.plmn_id: 999/70` |
| Visited AMF accepts roaming UE | `visited_network/configs/amf.yaml` | `access_control` includes `001/01` |
| Home SEPP local and N32 roles | `home_network/configs/sepp.yaml` | `sbi.server` and `n32.server` |
| Visited SEPP local and N32 roles | `visited_network/configs/sepp.yaml` | `sbi.server` and `n32.server` |
| SEPP dual-homing | `home_network/docker-compose.yaml`, `visited_network/docker-compose.yaml` | SEPP attached to local PLMN network and interconnect network |
| LBO session policy | `visited_network/configs/pcf.yaml` | `policy` for Home SUPI range |
| LBO session control | `visited_network/configs/smf.yaml` | UE session subnet and DNN |
| LBO user-plane anchor | `visited_network/configs/upf.yaml` | UE session subnet and `ogstun` |
| UE roaming identity | `packetrusher/config.example.yaml` | `ue.hplmn: 001/01` and IP points to VM's IP |

---

## 11. Repository Layout

```text
open5gs-sepp-roaming-single-vm/
├── README.md
├── medium-article.md
├── diagrams/
│   ├── high-level-architecture.mmd
│   |__ sepp-roaming-flow.mmd
├── home_network/
│   ├── docker-compose.yaml
│   ├── .env.example
│   ├── configs/
│   │   ├── nrf.yaml
│   │   ├── ausf.yaml
│   │   ├── udm.yaml
│   │   ├── udr.yaml
│   │   └── sepp.yaml
│   └── docker-open5gs/
├── visited_network/
│   ├── docker-compose.yaml
│   ├── .env.example
│   ├── configs/
│   │   ├── nrf.yaml
│   │   ├── amf.yaml
│   │   ├── smf.yaml
│   │   ├── upf.yaml
│   │   ├── pcf.yaml
│   │   └── sepp.yaml
│   └── docker-open5gs/
├── packetrusher/
│   ├── config.example.yaml
│   └── README.md
```
---

## 14. Expected Successful Behavior

When the lab is working, the expected behavior is:

| Area | Expected result |
|---|---|
| SEPP | N32 handshake between `v-SEPP` and `h-SEPP` |
| AMF | `v-AMF` detects Home PLMN `001/01` from the UE identity |
| Discovery | `v-AMF` discovers the Home authentication path through NRF/SEPP routing |
| Authentication | Home authentication flows through `h-AUSF`, `h-UDM`, and `h-UDR` |
| Registration | PacketRusher receives Registration Accept |
| PDU session | PacketRusher receives PDU Session Establishment Accept |
| UE address | UE receives an address from `10.45.0.0/16` |
| User plane | Traffic exits through `v-UPF` |

---

## 15. Troubleshooting by Concept

| Symptom | Conceptual meaning | Where to check |
|---|---|---|
| UE authenticates against Visited AUSF | UE is being treated as local, not roaming | PacketRusher `ue.hplmn` |
| MAC failure | UE credentials do not match Home subscriber | Home subscriber DB, PacketRusher K/OPc/SQN |
| SEPP wrong interface error | Peer traffic reached the SBI side | SEPP FQDNs, Docker aliases, `n32.server.address` |
| SEPP peer FQDN resolves but curl fails | SEPP is not listening on the target interface | `ss -ntlp` inside SEPP |
| Registration works but ping fails | Control plane works, user plane does not | `v-UPF`, `ogstun`, NAT, `gtp5g`, routes |
| PDU session fails | Session policy or UPF selection issue | `v-PCF`, `v-SMF`, `v-UPF` |

---

## 16. Scope and Limitations

This repository is an educational lab, not a production roaming deployment.

It does not attempt to fully model:

- IPX provider topology,
- production PKI and certificate lifecycle,
- production-grade SEPP security policy,
- separate physical PLMN sites,
- commercial roaming agreement enforcement,
- full N32-c/N32-f production separation,
- multi-tenant observability and operations.

The purpose is to isolate the most important roaming concepts in a reproducible single-VM environment.

---

## 18. References

- Open5GS roaming tutorial: <https://open5gs.org/open5gs/docs/tutorial/05-roaming/>
- Borijs131 - docker_open5gs: <https://github.com/Borjis131/docker-open5gs>
- 5G ROAMING (Open5GS, Packet Rusher): <https://medium.com/@vidime.sa.buduci.rok/5g-roaming-open5gs-packet-rusher-dacb34f3497c>

---
