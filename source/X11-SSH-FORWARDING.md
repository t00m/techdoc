---
Category: Procedure
Command: echo, sudo, xauth, xterm, systemctl, xterm, ssh, dnf
Date: 2025-12-15
DocType: How-to guide
EnvVar: $DISPLAY
Filename: sshd_config
Filesystem: /etc, /etc/ssh
OS: Linux, Windows
Priority: Normal
Product: MobaXterm, OpenSSH, Putty, Xming
Protocol: SSH, X11
Scope: Unix Administration
Service: sshd
Subsystem: Systemd
Status: Released
Tag: forwarding, graphical, display, remote, secure, client
Topic: Administration, Configuration, Connectivity
---
 
# How To Configure X11 Forwarding Using SSH In Linux
 
## Problem
 
When connecting to a remote Linux server via SSH, graphical applications cannot be displayed
locally unless X11 forwarding is properly configured on both the client and the server side.
 
## Prerequisites
 
- SSH access to the target Linux server.
- An X server running on the client machine (built-in on Linux/macOS; requires Xming or similar on Windows).
- Sufficient privileges to edit `/etc/ssh/sshd_config` on the server (or access to a sudoer account).
## Procedure
 
### Client
 
Ensure that the SSH client has the **X11 forwarding** option enabled before connecting.
 
- **MobaXterm / Linux SSH client:** X11 forwarding is typically enabled by default or via the session settings.
- **Putty:** Putty does not include a built-in X server. [Xming](https://sourceforge.net/projects/xming/) must be installed and running separately. Enable X11 forwarding under *Connection → SSH → X11 → Enable X11 forwarding*.
### Server
 
#### 1. Enable X11 Forwarding in SSHD
 
Edit `/etc/ssh/sshd_config` and ensure the following directive is present and not commented out:
 
```
X11Forwarding yes
```
 
Restart the SSH daemon after any change to this file:
 
```bash
sudo systemctl restart sshd
```
 
#### 2. Authorise the Target User
 
Once connected with X11 forwarding active, grant the target OS user access to the current X display:
 
```bash
sudo -u <os_user> xauth add `xauth list $DISPLAY`
```
 
#### 3. Verify the DISPLAY Variable
 
Confirm that the `DISPLAY` environment variable is correctly propagated for the target user:
 
```bash
sudo -u <os_user> echo $DISPLAY
```
 
Expected output example: `:10.0` or `localhost:10.0`.
 
#### 4. Test the Forwarded Display
 
Launch a minimal X application to confirm the full chain is working:
 
```bash
sudo -u <os_user> xterm
```
 
An `xterm` window should open on the local desktop.
 
## Verification
 
| Check | Expected Result |
|---|---|
| `X11Forwarding yes` present in `/etc/ssh/sshd_config` | Directive uncommented and set to `yes` |
| `echo $DISPLAY` for target user | Returns a non-empty value such as `localhost:10.0` |
| `xterm` launches | A terminal window appears on the local X display |
 
## Troubleshooting
 
- **`DISPLAY` is empty:** the SSH session was not started with X11 forwarding enabled. Reconnect using `ssh -X user@host` or enable the option in the client.
- **`xauth` command not found:** install the `xauth` package (`dnf install xorg-x11-xauth` on RHEL/Rocky).
- **`xterm` command not found:** install the `xterm` package (`dnf install xterm`).
- **Xming not displaying on Windows:** ensure Xming is started *before* opening the SSH session in Putty.
