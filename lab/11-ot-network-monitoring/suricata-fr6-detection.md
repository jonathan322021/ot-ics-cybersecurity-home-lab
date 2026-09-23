# OT Network Monitoring and Detection with Suricata



## Overview



This lab implements network-based intrusion detection in the OT/ICS home lab using Suricata on pfSense.



The objective is to demonstrate how network segmentation and network security monitoring provide complementary security controls.



The lab correlates an unauthorized Modbus TCP connection attempt from an IT asset with:



- a Suricata IDS alert;

- a pfSense firewall block event;

- the originating connection attempt from Kali Linux.



The exercise demonstrates concepts related to ISA/IEC 62443:



- FR5 — Restricted Data Flow

- FR6 — Timely Response to Events



---



## Lab Architecture



```text

&#x20;                        INTERNET

&#x20;                           |

&#x20;                      VMware NAT

&#x20;                           |

&#x20;                        pfSense

&#x20;                   +-------+-------+

&#x20;                   |               |

&#x20;                IT / LAN           OT

&#x20;            192.168.10.0/24   192.168.20.0/24

&#x20;                   |               |

&#x20;             +-----+-----+    +----+---------+

&#x20;             |           |    |              |

&#x20;       Windows Admin    Kali  HMI          OpenPLC

&#x20;       192.168.10.100   .50   .20            .30

&#x20;                                          TCP/502

```



Suricata is deployed on the pfSense LAN interface in IDS mode.



```text

Kali

192.168.10.50

&#x20;     |

&#x20;     | TCP SYN -> 192.168.20.30:502

&#x20;     v

Suricata IDS

&#x20;     |

&#x20;     | ALERT

&#x20;     v

pfSense Firewall

&#x20;     |

&#x20;     X BLOCK

&#x20;     |

OpenPLC

192.168.20.30

```



---



## Assets and Network Roles



| Asset | IP Address | Zone | Role |

|---|---|---|---|

| pfSense | 192.168.10.1 / 192.168.20.1 | Security Boundary | Firewall and IDS platform |

| Kali Linux | 192.168.10.50 | IT | Controlled security testing host |

| Windows Admin | 192.168.10.100 | IT | Authorized administration workstation |

| FactoryTalk Optix HMI | 192.168.20.20 | OT | Operator HMI |

| OpenPLC Runtime | 192.168.20.30 | OT | PLC runtime and Modbus TCP server |



---



## Security Objective



The intended security policy is:



```text

Windows Admin -> OpenPLC TCP/8443    ALLOW

Kali          -> OT Network          BLOCK

IT            -> OT Network          BLOCK by default

```



The monitoring objective is to detect an unauthorized attempt from Kali to access the OpenPLC Modbus TCP service.



Expected event:



```text

Source:       192.168.10.50

Destination:  192.168.20.30

Protocol:     TCP

Destination Port: 502

Firewall:     BLOCK

Suricata:     ALERT

```



---



## ISA/IEC 62443 Security Concepts



### FR5 — Restricted Data Flow



FR5 focuses on restricting communications between systems and zones according to the intended architecture and security policy.



In this lab, pfSense enforces the IT-to-OT boundary.



The existing firewall rule:



```text

BLOCK IT-LAN TO OT

```



prevents unauthorized IT systems from directly accessing OT assets.



### FR6 — Timely Response to Events



Prevention alone does not provide sufficient visibility into security events.



Suricata provides monitoring and alerting capabilities that allow unauthorized communication attempts to be identified and investigated.



In this lab:



```text

pfSense Firewall = Enforcement

Suricata IDS     = Detection

```



Together they provide complementary security controls.



---



## Suricata Deployment



Suricata was installed as a pfSense package and configured on:



```text

Interface: LAN (em1)

Description: OT-LAB-IT-MONITORING

Pattern Match: AUTO

Blocking Mode: DISABLED

```



Blocking was intentionally disabled.



The purpose of this phase is to use Suricata strictly as an IDS while pfSense remains responsible for network enforcement.



This separation makes it possible to distinguish detection from prevention during testing.



---



## IDS Rule Source



Emerging Threats Open rules were enabled as the base IDS ruleset.



The ruleset was successfully downloaded and verified through the Suricata update interface.



A custom rule was then created specifically for the OT lab.



---



## Custom OT Detection Rule



The following Suricata rule detects TCP SYN attempts from the Kali security testing host to the OpenPLC Modbus TCP service:



```text

alert tcp 192.168.10.50 any -> 192.168.20.30 502 (msg:"OT-LAB Unauthorized IT to OpenPLC Modbus TCP Attempt"; flags:S; sid:1000001; rev:1;)

```



### Rule Breakdown



| Field | Meaning |

|---|---|

| `alert` | Generate an IDS alert |

| `tcp` | Inspect TCP traffic |

| `192.168.10.50` | Kali source host |

| `any` | Any source port |

| `->` | Traffic direction |

| `192.168.20.30` | OpenPLC destination |

| `502` | Modbus TCP service |

| `flags:S` | Detect TCP SYN packets |

| `sid:1000001` | Local signature identifier |

| `rev:1` | Rule revision |



The SYN flag was selected because the firewall blocks the connection before a full TCP session can be established.



---



## Controlled Security Test



From Kali Linux, the following controlled connectivity test was executed:



```bash

nc -vz -w 3 192.168.20.30 502

```



Kali returned:



```text

192.168.20.30: inverse host lookup failed: Unknown host

(UNKNOWN) \[192.168.20.30] 502 (?) : Connection timed out

```



A timeout by itself does not prove that a firewall blocked the connection.



The result must be correlated with firewall and IDS telemetry.



\---



## Suricata Detection



Suricata generated alerts for the attempted connection.



Observed event:



```text

Source:       192.168.10.50

Destination:  192.168.20.30

Destination Port: 502

Protocol:     TCP

GID:SID:      1:1000001

```



Alert message:



```text

OT-LAB Unauthorized IT to OpenPLC Modbus TCP Attempt

```



Multiple alerts were observed because TCP retransmitted SYN packets while waiting for a response.



!\[Kali Modbus attempt and Suricata alert](../../screenshots/ot-network-monitoring/03-kali-modbus-attempt-suricata-alert.png)



---



## Firewall Correlation



The same communication attempt was observed in the pfSense firewall logs.



The firewall recorded:



```text

Interface:    LAN

Rule:         BLOCK IT-LAN TO OT

Source:       192.168.10.50

Destination:  192.168.20.30:502

Protocol:     TCP:S

Action:       BLOCK

```



The events occurred at matching timestamps:



```text

11:20:25

11:20:26

11:20:27

```



!\[pfSense firewall correlation](../../screenshots/ot-network-monitoring/04-pfsense-firewall-correlation.png)



---



## Event Correlation



The complete event chain was:



```text

Kali

192.168.10.50

&#x20;     |

&#x20;     | TCP SYN :502

&#x20;     v

Suricata

SID 1000001

&#x20;     |

&#x20;     | ALERT

&#x20;     v

pfSense Firewall

BLOCK IT-LAN TO OT

&#x20;     |

&#x20;     X

&#x20;     |

OpenPLC

192.168.20.30

```



The same event was therefore visible at three levels:



1\. Source host — connection attempt.

2\. IDS — security alert.

3\. Firewall — enforcement action.



This correlation provides stronger evidence than relying on a single telemetry source.



---



## Detection vs Enforcement



This exercise demonstrates an important security distinction.



### Detection



Suricata answers:



```text

What security-relevant activity is occurring?

```



Result:



```text

ALERT

```



\### Enforcement



pfSense answers:



```text

Is this communication permitted by policy?

```



Result:



```text

BLOCK

```



The IDS does not replace the firewall, and the firewall does not replace monitoring.



---



## Security Findings



### Finding 1 — Unauthorized IT-to-OT Modbus traffic is blocked



The existing segmentation policy successfully prevented Kali from reaching the OpenPLC Modbus TCP service.



### Finding 2 — Unauthorized attempts are detectable



Suricata successfully generated an alert for the prohibited communication attempt.



### Finding 3 — IDS and firewall telemetry can be correlated



Source address, destination address, destination port, protocol, and timestamps matched across the Suricata and pfSense evidence.



### Finding 4 — TCP retransmissions create multiple observable events



Because the firewall silently dropped the TCP SYN, the client retransmitted the connection request.



These retransmissions were visible in both IDS and firewall telemetry.



---



## ISA/IEC 62443 Mapping



| Control Area | Lab Implementation |

|---|---|

| FR5 — Restricted Data Flow | pfSense IT-to-OT firewall policy |

| FR6 — Timely Response to Events | Suricata IDS alerts |

| Zones | IT and OT networks |

| Conduit | pfSense-controlled communication path |

| Least Privilege | Only explicitly authorized IT-to-OT communication is permitted |

| Monitoring | Suricata inspection on LAN |

| Event Correlation | Suricata alerts correlated with pfSense firewall logs |



---



## Limitations and Architectural Observation



The FactoryTalk Optix HMI and OpenPLC currently reside on the same OT subnet:



```text

HMI      192.168.20.20

OpenPLC  192.168.20.30

```



Because both systems share the same Layer 2 network, their direct HMI-to-PLC communication does not traverse pfSense.



Therefore, the current Suricata instance on the pfSense LAN interface cannot provide complete visibility into direct HMI-to-PLC traffic.



A future lab phase will address this limitation using improved OT zone segmentation and/or passive monitoring through network traffic mirroring.



---



## Lessons Learned



This exercise demonstrated that:



- Firewall enforcement and IDS detection solve different security problems.

- A blocked connection attempt can still provide valuable security telemetry.

- A TCP timeout alone is not proof of firewall enforcement.

- Security events should be correlated across multiple telemetry sources.

- TCP retransmissions explain repeated alerts for the same connection attempt.

- Custom IDS signatures can encode expected OT communication policy.

- Monitoring architecture depends heavily on network topology and traffic visibility.

- ISA/IEC 62443 FR5 and FR6 can be implemented as complementary controls.



---



## Result



The test successfully demonstrated:



```text

Unauthorized IT -> OT Modbus attempt

&#x20;            |

&#x20;            +--> Suricata ALERT

&#x20;            |

&#x20;            +--> pfSense BLOCK

```



The lab now provides both preventive and detective controls for the tested IT-to-OT communication path.



This establishes a foundation for future OT security monitoring exercises using tools such as Zeek, Security Onion, centralized logging, and additional industrial protocol detections.

