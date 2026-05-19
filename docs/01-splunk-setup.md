# Milestone 1: Splunk Setup

## Objective

Set up Splunk Free on a dedicated Ubuntu Server VM to serve as the SIEM for the home SOC lab. Splunk will ingest logs from a Windows endpoint in the next milestone and provide the platform
for writing detection queries against simulated attacks.

## Architecture Decision

Splunk runs on its own VM rather than alongside other services.
Reasoning:

- Keeps the SIEM isolated from the systems it monitors
- Mirrors real-world SOC architecture where the SIEM sits separately from monitored endpoints
- Allows independent snapshotting and rollback if something breaks

## Environment

- **Host:** Windows, 32 GB RAM
- **Hypervisor:** VMware Workstation Pro 17
- **VM specs:**
  - OS: Ubuntu Server 24.04 LTS
  - CPU: 2 cores
  - RAM: 4 GB
  - Disk: 80 GB
  - Network: NAT
- **VM IP:** 192.168.68.130
- **Splunk version:** 9.4.x (Free)

## Setup Steps

### 1. VM Creation

Created the VM in VMware Workstation Pro with the specs above. Removed unused hardware (sound, printer, USB) to keep the VM lean. ISO stored at `D:\VMs\ISO Images\`.

### 2. Ubuntu Server Installation

Installed Ubuntu Server 24.04 LTS. Key choices:

- Used the entire disk with LVM
- Created admin user `charles`
- Installed OpenSSH server so the VM could be managed from the Windows host instead of through the VMware console
- Skipped Ubuntu Pro and the featured snaps to keep the install minimal

### 3. Post-Install Verification

After first boot, confirmed the system was healthy:

```
ip a            # confirmed DHCP-assigned IP: 192.168.68.130
ping google.com # confirmed internet access
df -h           # confirmed disk space
sudo apt update
sudo apt upgrade -y
```

### 4. SSH from Windows Host

Connected to the VM from Windows Terminal via SSH:

```
ssh charles@192.168.68.130
```

All further administration of the VM was done over SSH rather than the VMware console. Real admin practice and a much cleaner workflow than typing inside the VMware window.

### 5. Splunk Installation

Downloaded the Splunk Enterprise `.deb` package from Splunk's website (free account required to access downloads). Transferred to the VM with `wget`, then installed:

```
sudo dpkg -i splunk.deb
ls /opt/splunk    # confirmed install in /opt/splunk
```

### 6. First Startup and Admin User Creation

This is where the install got interesting and I had to iterate.

First attempt:

```
sudo /opt/splunk/bin/splunk start --accept-license
```

Splunk refused to run as root by default. Production note: in a real SOC environment, Splunk should run as a dedicated `splunk` system user, not root. For a home lab this is
acceptable; in production I would create the user, change file ownership with `chown`, and harden the service.

Used the `--run-as-root` flag to proceed:

```
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

During initial setup, I made a mistake — typed my intended password as the username during the admin account creation prompt. This left Splunk with broken admin credentials.

### 7. Admin User Recovery

Tried several recovery approaches:

```
# Attempt 1: edit user with default credentials — failed because
# the existing admin user was broken
sudo /opt/splunk/bin/splunk edit user admin -password NewPass -auth admin:admin

# Result: "could not get info for non-existent user='admin'"
```

The clean approach: stop Splunk, delete the existing user database, and re-seed on startup.

```
sudo /opt/splunk/bin/splunk stop
sudo rm -f /opt/splunk/etc/passwd
```

Created a `user-seed.conf` file that tells Splunk to create the admin user on next startup:

```
echo -e "[user_info]\nUSERNAME = admin\nPASSWORD = <password>" \
  | sudo tee /opt/splunk/etc/system/local/user-seed.conf
```

Then started Splunk fresh:

```
sudo /opt/splunk/bin/splunk start
```

Splunk read the seed file on startup, created the admin user, and reported the web UI was available.

### 8. Web UI Access

Confirmed the Splunk web UI was reachable from the Windows host by navigating to `http://192.168.68.130:8000` in a browser.

Logged in with the admin user and password from the seed file. Splunk home screen loaded — search apps, dashboards menu, ready for data ingestion.

## VM Snapshot

After confirming Splunk was running and accessible, took a snapshot in VMware named `M1 - Splunk installed and running`. This is a clean rollback point if future milestones break the environment.

## What Went Wrong / What I Learned

- **Splunk's default startup behaviour blocks root execution.** This is a security feature, not a bug. Used `--run-as-root` for the home lab; in production I would create a dedicated
  `splunk` user.
- **The admin account creation prompt is unforgiving.** Once you type the wrong value, recovery requires resetting the user database — there's no graceful retry.
- **The user-seed.conf approach is the cleanest way to reset admin credentials.** Faster and more reliable than the `edit user -auth` route, which requires working credentials
  to start with.
- **Documentation while building > documentation after.**
  Notes taken during the install captured details I would have forgotten by the end of the milestone — like the exact error message and which flag was needed.

## What's Next

Milestone 2: install a Windows 10 VM, configure Sysmon for detailed endpoint telemetry, and forward Windows logs to Splunk using the Splunk Universal Forwarder.

## Screenshots

- `screenshots/01-splunk-login.png` — Splunk login page reached from the Windows host
- `screenshots/02-splunk-home.png` — Splunk home screen after successful admin login
