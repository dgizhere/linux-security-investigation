# Linux Security Investigation

## 1. Objective

The objective of this project was to investigate the security state of a Linux system by examining network services, running processes, system services, installed packages, and AppArmor security controls.

---

## 2. Environment

I created a virtual lab machine using VMware and installed Ubuntu 26.04.1 LTS. I performed the investigation in this controlled lab environment, focusing on the CUPS service, its associated processes and network sockets, systemd service configuration, package attribution, and AppArmor security restrictions.

---

## 3. Investigation Methodology

First, I checked socket statistics using `ss` to identify listening ports and the processes associated with them. After identifying the process, I investigated its parent process and then examined the CUPS service associated with it. I then reviewed the relevant system logs to understand what had happened. Finally, I investigated the AppArmor restrictions and the denied operations associated with the CUPS process.

---

## 4. Network Service Investigation

I used the following command to identify listening TCP sockets, numeric addresses and ports, and associated processes:

```bash
ss -ltnp
```

The investigation identified the following listener:

```text
127.0.0.1:631
```

The associated process was:

```text
cupsd
PID 1569
```

Port 631 was associated with the CUPS printing service.

The service was bound to the loopback address `127.0.0.1`. This means the listener was bound to the local machine rather than directly exposed through the machine's network interfaces.

### Finding

The CUPS service was listening locally on TCP port 631. The listener was not bound to all IPv4 interfaces.

---

## 5. Process Investigation

I investigated PID `1569` using:

```bash
ps -p 1569 -o pid,ppid,user,%cpu,%mem,etime,args
```

The process information showed:

| Field                | Value                |
| -------------------- | -------------------- |
| PID                  | 1569                 |
| PPID                 | 1                    |
| User                 | root                 |
| CPU                  | 0.0%                 |
| Memory               | 0.3%                 |
| Executable/Arguments | `/usr/sbin/cupsd -l` |

PID 1 was identified as `systemd`, indicating that the CUPS process was ultimately managed through the system service manager.

The executable associated with the process was:

```text
/usr/sbin/cupsd
```

The process was running as root. Its resource usage at the time of investigation was low.

### Finding

The process was consistent with a system-managed CUPS daemon. Running as root was noted as a security-relevant property, but it was not treated as evidence of compromise by itself.

---

## 6. Service and Package Investigation

I checked the current state of the CUPS service using:

```bash
sudo systemctl status cups
```

This confirmed that the CUPS service was active and running and identified PID `1569` as its main process.

I then used:

```bash
sudo dpkg -S /usr/sbin/cupsd
```

to identify which installed package provided the executable.

The result showed:

```text
cups-daemon: /usr/sbin/cupsd
```

I then performed a separate package verification check:

```bash
sudo dpkg -V cups-daemon
```

The command produced no output, meaning no discrepancies were reported by that verification.

### Finding

The CUPS executable was associated with the installed `cups-daemon` package, and the package verification did not report any discrepancies.

---

## 7. AppArmor Investigation

### 7.1 AppArmor Denials

Relevant system logs showed AppArmor `DENIED` events associated with the `cupsd` process.

One event showed that `cupsd` attempted to read:

```text
/etc/paperspecs
```

and AppArmor denied the read operation.

Another event showed that `cupsd` attempted to use:

```text
CAP_NET_ADMIN
```

and AppArmor denied the capability request.

These events demonstrated that AppArmor was actively restricting specific operations attempted by the CUPS process.

### 7.2 AppArmor Policy Investigation

I inspected the AppArmor profile for `cupsd`:

```text
/etc/apparmor.d/usr.sbin.cupsd
```

The profile contained the following rule:

```text
/etc/paperspecs r,
```

This appeared to permit read access to `/etc/paperspecs`, while the system log showed that a read attempt was denied.

I investigated the profile, AppArmor enforcement state, package information, and relevant logs further. The exact reason for the policy/runtime discrepancy was not conclusively established during this investigation.

### 7.3 Assessment

AppArmor blocked specific operations attempted by `cupsd`. The investigation found no evidence of compromise in the checks performed, but the `/etc/paperspecs` policy/runtime discrepancy was not conclusively explained.

---

## 8. Findings

The investigation produced the following findings:

### Finding 1 — CUPS listener

CUPS was listening on TCP port `631` through the loopback address:

```text
127.0.0.1:631
```

The service was therefore locally accessible rather than directly exposed through all network interfaces.

### Finding 2 — CUPS process

The listener was associated with:

```text
cupsd
PID 1569
```

The process was running as root and was managed through the system service infrastructure.

### Finding 3 — Package attribution

The executable:

```text
/usr/sbin/cupsd
```

was provided by the `cups-daemon` package.

Package verification reported no discrepancies.

### Finding 4 — AppArmor restrictions

AppArmor was enforcing restrictions on `cupsd` and blocked specific operations, including an attempted read of `/etc/paperspecs` and an attempted use of `CAP_NET_ADMIN`.

### Finding 5 — Unresolved discrepancy

The AppArmor profile contained a rule permitting read access to `/etc/paperspecs`, while the logs showed a denied read operation. The exact reason for this discrepancy was not conclusively determined.

---

## 9. Security Assessment

Based on the evidence collected during this investigation, no evidence of compromise was found.

The CUPS process, service configuration, executable location, package ownership, and package verification were consistent with a legitimate system service. The service was also bound to the loopback interface rather than all network interfaces.

AppArmor was enforcing security restrictions and successfully blocked specific operations attempted by `cupsd`.

However, the `/etc/paperspecs` policy/runtime discrepancy remained unresolved. Therefore, the investigation does not claim that the system is guaranteed to be uncompromised; it only records that no evidence of compromise was identified during the checks performed.

---

## 10. Conclusion

This investigation demonstrated a basic Linux security investigation workflow:

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

The investigation showed how a potentially interesting listening service can be traced through the operating system to determine what process is responsible, how that process is managed, which package provides it, and how security controls such as AppArmor interact with it.

The investigation did not identify evidence of compromise, although one AppArmor policy/runtime discrepancy remained unresolved.
