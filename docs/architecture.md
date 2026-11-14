# Architecture & Topology

## Overview

This hybrid Wazuh SOC lab integrates a virtualized Linux management plane with a Windows 11 host agent to simulate security monitoring, log collection, custom rule correlation, and automated containment.

## Topology Diagram

```text
+-------------------------------------------------------------+
|                       Windows 11 Host                       |
|            (Wazuh Agent + Windows Security Logs)            |
+-------------------------------------------------------------+
                               |
                               | VirtualBox NAT / Port Forwarding
                               | (Host Port 2222 -> Guest Port 22)
                               v
+-------------------------------------------------------------+
|                       Ubuntu Linux VM                       |
|       (Wazuh Manager + OpenSearch Indexer + Dashboard)      |
+-------------------------------------------------------------+

## Port Forwarding & Communication Table

| Service / Purpose | Host Port | Guest (VM) Port | Description |
|---|---|---|---|
| SSH Access | 2222 | 22 | Management access to Ubuntu VM and attack simulation vector |
| Wazuh Dashboard | 4433 | 443 | HTTPS Web UI for threat hunting, dashboards, and alerts |
| Wazuh Agent Enrollment | 1515 | 1515 | Control channel used by agents to register with the manager |
| Wazuh Agent Connection | 1514 | 1514 | Encrypted TCP/UDP telemetry stream from agent to manager |
| Wazuh RESTful API | *Internal* | 55000 | Internal communication channel between Dashboard and Manager |