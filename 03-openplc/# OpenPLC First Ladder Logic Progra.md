# OpenPLC First Ladder Logic Program

## Objective

Create and test a basic PLC program using OpenPLC Editor.

## Architecture

Engineering Workstation:
192.168.10.100

OpenPLC Runtime:
192.168.20.30

Management Protocol:
HTTPS TCP/8443

## Ladder Logic

START controls MOTOR.

START = FALSE -> MOTOR = FALSE
START = TRUE  -> MOTOR = TRUE

## ISA/IEC 62443 Concepts

- Asset
- Engineering Workstation
- System Integrity
- Restricted Data Flow
- Least Privilege
- Zones and Conduits

## Security Relevance

PLC control logic directly influences the industrial process.
Unauthorized modification of PLC logic could therefore impact
process integrity and availability.

## Verification

The OpenPLC debugger was used to verify:

START FALSE -> MOTOR FALSE
START TRUE  -> MOTOR TRUE
