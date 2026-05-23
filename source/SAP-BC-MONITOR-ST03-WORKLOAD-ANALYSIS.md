---
Author: Tomás Vírseda
Category: Report
Date: 2026-05-23
DocType: Reference
OS: Linux
Product: SAP Netweaver, SAP GUI
Protocol: HTTP, HTTPS, NNTP, SMTP, FTP, RFC, CPIC, LCOM
Scope: Database Administration, SAP Basis
Tag: workload, response time, wait time, work process, memory, bottleneck, buffer, network, slow
Topic: Monitoring, Performance, Analysis, Troubleshooting
Transaction: SM50, SM13, SM12, RZ04, RZ10, RZ11, ST02, ST06, DB02, DB04, ST05
---

# ST03: SAP Workload Analysis

## Overview

Transaction **ST03** (Workload Monitor) provides a breakdown of the time components that make up
the total response time for ABAP work processes. Understanding each component - and its acceptable
threshold - is essential for diagnosing performance issues in an SAP system.

## Time Components

### Wait Time

**Definition:** Queue time for a Dialog (DIA) work process to become available (visible in SM50).

| Attribute | Detail |
|---|---|
| Threshold | < 50 ms |
| Indicates | Insufficient work processes, inactive update task, incorrect operation mode, locked tasks, long-running transactions |
| Resolution | Ensure extended memory is sufficient before adding more work processes. |
| Transactions | SM50, RZ04, RZ10, RZ11, SM13, SM12 |

### Roll In / Roll Out Time

**Definition:** Time to copy the user context (internal tables, parameters, screen lists) to/from
work process memory.

| Attribute | Detail |
|---|---|
| Threshold | < 20 ms |
| Indicates | Undersized SAP roll buffer or extended memory; CPU bottleneck |
| Resolution | Tune roll memory and extended memory parameters. |
| Transactions | RZ10, RZ11, ST02, ST06 |

### Load Time

**Definition:** Time to load and generate programs and screens from SAP buffers.

| Attribute | Detail |
|---|---|
| Threshold | < 50 ms |
| Indicates | Undersized program/CUA/screen buffer, recent transports to production, CPU bottleneck |
| Resolution | Tune SAP buffers. |
| Transactions | RZ10, RZ11, ST02, ST06 |

### Response Time

**Definition:** Total elapsed time from when a request arrives in the system until processing is
complete.

```
Response Time = Roll In + Wait + Load/Gen + DB Time + Processing + Roll Wait + Enqueue + (other)
```

This is the top-level KPI. All other time components are sub-components of it.

### Processing Time

**Definition:** CPU-bound time consumed by ABAP processing.

```
Processing Time = Response Time - Wait Time - Load/Gen Time - DB Time - Roll In Time - Enqueue Time - DB Procedure Time
```

| Attribute | Detail |
|---|---|
| Threshold | Processing time ≤ 2× CPU time |
| Indicates | If Processing >>> 2× CPU time → CPU bottleneck |

### DB Time

**Definition:** Time spent reading from or writing to the database.

| Attribute | Detail |
|---|---|
| Threshold | < 40% of total response time |
| Indicates | Poorly tuned database, I/O disk problems, slow/expensive SQL statements |
| Resolution | Check OS, storage, and DB layer. For custom (Z) reports with expensive SQL, coordinate with developers to optimize queries. |
| Transactions | ST06, DB02, DB04, ST05 |

### Roll Wait Time

**Definition:** Time waiting for a reply from RFC connections.

| Attribute | Detail |
|---|---|
| Threshold | < 200 ms |
| Indicates | Slow network or slow remote system |

### GUI Time

**Definition:** Time for data transfer from the SAP application server to the SAPGUI workstation,
including LAN/WAN roundtrips.

| Attribute | Detail |
|---|---|
| Threshold | No fixed threshold; compare against baseline |
| Indicates | Network performance issues (WAN/VPN) or slow client PC |

### Enqueue Time

**Definition:** Time for a work process to set an enqueue (row lock).

| Attribute | Detail |
|---|---|
| Threshold | Should be negligible |
| Indicates | Lock contention or enqueue server issues |

---

## Task Types

The task type identifies the nature of the work process activity recorded in each statistics record.

| Task Type | Description |
|---|---|
| `AUTOABAP` | Automatically-processed report (e.g. monitoring tools) |
| `B.INPUT` | Transaction step in batch input mode, processed by a Dialog work process |
| `BACKGROUND` | Transaction step started by a background processing work process |
| `BUFFER SYNC` | Periodic local table buffer synchronization (controlled by `rdisp/bufreftime`) |
| `DIALOG` | Standard online transaction step initiated by a user |
| `RFC` | Remote Function Call in the ABAP system, processed by a Dialog work process |
| `CPIC` | Communication via CPIC interface, processed by a Dialog work process |
| `SPOOL` | Transaction step of a spool work process |
| `UPDATE` | Transaction step of the SAP update task (V1), started by the dispatcher |
| `UPDATE2` | V2 (asynchronous) update |
| `ALE` | IDoc processing, handled by a Dialog work process |
| `LCOM` | Fast RFC (fRFC/LCOM-RFC) using shared memory pipeline (ABAP ↔ Java in Web AS only) |
| `HTTP`, `HTTPS`, `NNTP`, `SMTP`, `FTP` | ICM requests based on the corresponding Internet protocol |
| `ENQUEUE` | Enqueue handler task |
| `DIALOG(-)GUI` | Dialog task without GUI involvement |

---

## Quick Diagnostic Reference

| Symptom | Likely Component | First Transactions |
|---|---|---|
| High total response time | Investigate all sub-components | ST03, SM50 |
| Users waiting for work process | High Wait Time | SM50, RZ04, RZ10 |
| Memory-related slowness | High Roll In/Out | ST02, RZ10, RZ11 |
| Slow after transport | High Load Time | ST02, RZ10 |
| Database bottleneck | High DB Time | ST05, DB02, DB04 |
| Remote system slowness | High Roll Wait Time | SM59, network checks |
| GUI lag, WAN issues | High GUI Time | Network team |

---

## References

### SAP Help

- [Workload Overview](https://help.sap.com/saphelp_nw70/helpdata/en/21/2c8f38c7215428e10000009b38f8cf/content.htm)
- [Response Times: Rough Guide](https://help.sap.com/saphelp_nw70/helpdata/en/c4/3a6af1505211d189550000e829fbbd/content.htm)

### SAP Notes

- **SAP Note 8963** - Definition of SAP response time / processing time / CPU time
- **SAP Note 376148** - Response times without GUI time
- **SAP Note 364625** - Interpretation of response time in 4.6

### External

- [Analysing High SAP Roll Wait Time - NW702](http://darrylgriffiths.blogspot.com.es/2014/03/analysing-high-sap-roll-wait-time-nw702.html)
