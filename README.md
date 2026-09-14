# Linux Security Investigation

A hands-on Linux security investigation performed in a controlled VMware laboratory environment.

## Overview

This project demonstrates a basic Linux security investigation workflow by tracing a network listener to its process, service, executable, package, logs, and security policy.

The investigation focused on the **CUPS printing service (`cupsd`)** and its interaction with **systemd and AppArmor**.

## Objectives

* Identify listening network services.
* Identify the processes associated with listening ports.
* Investigate process ownership and parent processes.
* Identify the system service responsible for a process.
* Trace an executable back to its installed package.
* Review relevant system logs.
* Investigate AppArmor security restrictions.
* Assess whether the observed activity provided evidence of compromise.

## Lab Environment

* **Virtualization:** VMware
* **Operating System:** Ubuntu 26.04.1 LTS
* **Environment:** Controlled personal cybersecurity laboratory

## Investigation Workflow

```text
Network Socket
      ↓
Process
      ↓
Parent Process
      ↓
Service
      ↓
Executable
      ↓
Package
      ↓
Logs
      ↓
Security Policy
```

## Tools and Commands

The investigation used Linux tools including:

```bash
ss
ps
systemctl
journalctl
dpkg
aa-status
apparmor_parser
```

## Key Findings

### CUPS Network Listener

CUPS was found listening on TCP port `631` through:

```text
127.0.0.1:631
```

The listener was associated with the `cupsd` process.

Because the service was bound to the loopback interface, it was locally accessible rather than directly exposed through all network interfaces.

### Process Investigation

The CUPS daemon was identified as:

```text
PID: 1569
PPID: 1
USER: root
Executable: /usr/sbin/cupsd
```

PID 1 was identified as `systemd`, showing that the process was associated with the system service infrastructure.

### Package Investigation

The executable:

```text
/usr/sbin/cupsd
```

was attributed to the installed:

```text
cups-daemon
```

package.

Package verification reported no discrepancies.

### AppArmor Investigation

AppArmor was enforcing restrictions on `cupsd`.

The logs showed denied operations involving:

* Reading `/etc/paperspecs`
* Requesting `CAP_NET_ADMIN`

The AppArmor profile contained a rule allowing read access to `/etc/paperspecs`, while the log showed a denied read attempt. The exact reason for this policy/runtime discrepancy was not conclusively established.

## Security Assessment

No evidence of compromise was found during the checks performed.

The CUPS process, service configuration, executable location, package attribution, and package verification were consistent with a legitimate system service.

The investigation also confirmed that AppArmor was actively enforcing security restrictions.

However, the `/etc/paperspecs` policy/runtime discrepancy remained unresolved, so the investigation does not claim absolute certainty that the system was uncompromised.

## Evidence

Supporting evidence from the investigation is stored in the [`evidence`](./evidence/) directory.

* [Network Services](./evidence/network-services.txt)
* [Process Analysis](./evidence/process-analysis.txt)
* [Systemd Analysis](./evidence/systemd-analysis.txt)
* [AppArmor Analysis](./evidence/apparmor-analysis.txt)

## Detailed Report

For the complete investigation and assessment, see:

[Investigation Report](./investigation-report.md)

## Disclaimer

This project was performed on a personally controlled virtual machine for educational and cybersecurity training purposes.

No unauthorized systems were targeted.
