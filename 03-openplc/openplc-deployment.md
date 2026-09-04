# OpenPLC Runtime v4 – First PLC Deployment



\## Objective



Deploy and execute the first Ladder Logic program from the

Windows Engineering Workstation to the OpenPLC Runtime located

inside the OT network.



The objective is to maintain controlled IT-to-OT communication

through pfSense instead of providing unrestricted access between

the IT and OT zones.



\---



\## Architecture



Windows Engineering Workstation

192.168.10.100

&#x20;       |

&#x20;       | HTTPS TCP/8443

&#x20;       |

&#x20;       v

&#x20;    pfSense

&#x20;       |

&#x20;       | Controlled IT-to-OT Conduit

&#x20;       |

&#x20;       v

OpenPLC Runtime v4

192.168.20.30

&#x20;       |

&#x20;       v

OT\_Lab\_First\_PLC



\---



\## ISA/IEC 62443 Concepts



The exercise demonstrates several ISA/IEC 62443 concepts:



\- Zones

\- Conduits

\- Least Privilege

\- Network Segmentation

\- Restricted Data Flow (FR5)

\- System Integrity (FR3)

\- Defense in Depth



The Windows Engineering Workstation is located in the IT zone,

while the OpenPLC Runtime is located in the OT zone.



Communication between these zones must traverse pfSense.



Only the required management communication is permitted:



Source:

192.168.10.100



Destination:

192.168.20.30



Protocol:

TCP



Destination Port:

8443



All other general IT-to-OT traffic remains blocked.



\---



\## Initial Problem



The OpenPLC Runtime was running successfully on the OT system:



192.168.20.30



The service was listening on:



TCP/8443



However, the Windows Engineering Workstation could not connect.



Test:



Test-NetConnection 192.168.20.30 -Port 8443



Result:



TcpTestSucceeded : False



\---



\## Troubleshooting Methodology



The traffic path was analyzed hop-by-hop.



Windows

&#x20;  |

&#x20;  v

VMnet2

&#x20;  |

&#x20;  v

pfSense LAN

&#x20;  |

&#x20;  v

pfSense OT

&#x20;  |

&#x20;  v

OpenPLC



\### Windows Verification



The Windows workstation used:



SourceAddress:

192.168.10.100



Interface:

VMware Network Adapter VMnet2



This confirmed that Windows was using the correct IT interface.



\### OpenPLC Packet Capture



A packet capture was started on the OpenPLC host:



sudo tcpdump -ni ens33 'host 192.168.10.100 and tcp port 8443'



No packets were observed.



This demonstrated that the TCP SYN packets were not reaching

the OpenPLC host.



\### pfSense Packet Capture



A packet capture was then performed on the pfSense LAN interface.



Traffic was observed:



192.168.10.100:<ephemeral-port> -> 192.168.20.30:8443



Multiple attempts were visible.



This proved that:



Windows -> pfSense LAN = Working



but:



pfSense -> OpenPLC = Blocked



\---



\## Root Cause



The pfSense LAN firewall policy contained a general rule:



BLOCK LAN -> OT



The existing SSH exception only allowed:



192.168.10.100 -> 192.168.20.10 TCP/22



There was no exception for OpenPLC TCP/8443.



Because pfSense evaluates interface rules according to their

configured order, the OpenPLC traffic matched the LAN-to-OT

blocking rule.



\---



\## Remediation



A specific firewall rule was created before the general

LAN-to-OT block rule.



PASS



Source:

192.168.10.100



Destination:

192.168.20.30



Protocol:

TCP



Destination Port:

8443



Description:



ADMIN-WINDOWS -> OPENPLC HTTPS 8443



The resulting policy follows the principle of Least Privilege.



\---



\## Verification



The connectivity test was repeated:



Test-NetConnection 192.168.20.30 -Port 8443



Result:



TcpTestSucceeded : True



OpenPLC Editor was then configured with:



Device:

OpenPLC Runtime v4



IP Address:

192.168.20.30



The Editor successfully established a connection with the

remote runtime.



\---



\## First PLC Program



Project:



OT\_Lab\_First\_PLC



Language:



Ladder Diagram (LD)



Logic:



&#x20;      START             MOTOR

\--------| |---------------( )--------



Expected behavior:



START = FALSE -> MOTOR = FALSE



START = TRUE -> MOTOR = TRUE



\---



\## Build and Deployment



The OpenPLC Editor option used was:



Build and upload



The console confirmed:



Compilation completed successfully (exit code: 0)

PLC started.

Upload complete.

Compilation complete.



The Runtime status changed to:



PLC: RUNNING



Scan Cycle Statistics also confirmed that the PLC execution

cycle was running.



\---



\## Runtime Debugging



The OpenPLC Editor debugger was used to monitor the Ladder Logic.



START was changed to TRUE.



The Ladder rung became energized, demonstrating that the deployed

program was executing successfully on the remote OpenPLC Runtime.



\---



\## Security Finding



The original connectivity failure was not caused by OpenPLC.



The firewall was correctly enforcing the IT-to-OT segmentation

policy.



Instead of disabling segmentation, a specific authorized

communication flow was added.



Authorized Flow:



192.168.10.100

&#x20;       |

&#x20;       | TCP/8443

&#x20;       v

192.168.20.30



This represents a controlled communication conduit between the

Engineering Workstation and the PLC environment.



\---



\## Lessons Learned



1\. Verify the traffic path before modifying firewall rules.



2\. Packet captures should be performed at multiple points in the

&#x20;  communication path.



3\. A listening application does not prove that the network path

&#x20;  to the application is available.



4\. Firewall exceptions should be specific to source,

&#x20;  destination, protocol, and port.



5\. IT-to-OT communication should not be opened broadly just to

&#x20;  solve connectivity problems.



6\. OpenPLC Runtime v4 can be remotely managed from OpenPLC Editor

&#x20;  through TCP/8443.



7\. Successful network connectivity does not prove that the PLC

&#x20;  application is running; runtime status and PLC scan execution

&#x20;  must also be verified.



\---



\## Result



Engineering Workstation -> pfSense -> OpenPLC Runtime



TCP/8443: PASS



OpenPLC Editor connection: PASS



PLC program compilation: PASS



PLC program upload: PASS



PLC Runtime status: RUNNING



Ladder Logic execution: VERIFIED

