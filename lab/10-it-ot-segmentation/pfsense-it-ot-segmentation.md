# IT/OT Network Segmentation with pfSense



## Objective



The objective of this lab was to implement and validate restricted communication between the IT and OT networks using pfSense.



The security policy was designed according to the principle of least privilege:



- An authorized Windows administrative workstation is allowed to access the OpenPLC Runtime management service on TCP port 8443.

- A Kali Linux workstation located in the IT network is not allowed to access the OpenPLC Modbus TCP service on TCP port 502.

- General IT-to-OT communication is blocked unless explicitly authorized.



This lab provides practical evidence of network segmentation and restricted data flow in an OT/ICS environment.



---



## ISA/IEC 62443 Security Concept



This exercise is primarily related to:



### FR5 - Restricted Data Flow



ISA/IEC 62443 uses zones and conduits as part of the system security architecture.



A zone groups assets with similar security requirements, while a conduit represents controlled communication between zones.



In this lab:



- The IT network represents the IT Zone.

- The OT network represents the OT Zone.

- pfSense controls communication between the two networks.

- Specific firewall rules determine which traffic is allowed to cross the IT/OT boundary.



The security objective is not simply to provide network connectivity. Communication between zones should be restricted to required and authorized flows.



---



## Lab Architecture



```text


                  IT ZONE

               192.168.10.0/24



#x20;       +-------------------------------+

#x20;       |                               |

#x20;       |                               |

Windows Admin                       Kali Linux

192.168.10.100                    192.168.10.50

#x20;       |                               |

#x20;       +---------------+---------------+

#x20;                       |

#x20;                       |

#x20;                    pfSense

#x20;                LAN: 192.168.10.1

#x20;                       |

#x20;                Firewall Policy

#x20;                       |

#x20;                       |

#x20;                    OT ZONE

#x20;                192.168.20.0/24

#x20;                       |

#x20;                       |
#x20;                    OpenPLC

#x20;                 192.168.20.30

#x20;                  /          \\

#x20;             TCP/8443       TCP/502

#x20;             Management     Modbus TCP

