# Experiments

Reproducible Linux and networking experiments live here.

## Expectations

Each experiment should include a hypothesis, setup, steps, and results.

---

## Session 1: Fundamentals and Service Management (May 25, 2026)

### Experiment 1: File Creation and Basic Filesystem Operations

**Objective**: Understand basic filesystem operations and navigation.

**Hypothesis**: Files created with `touch` have default permissions that don't include execute. Files cannot be executed without the execute bit.

**Setup**:

```bash
mkdir devops
cd devops
```

**Steps**:

1. Create file: `touch app.log`
2. Write to file: `echo "hello linux" > app.log`
3. Read file: `cat app.log`

**Results**:

```
✓ Directory created successfully
✓ File created and written to
✓ Content readable
hello linux
```

**Observations**:

- Touch creates file in working directory
- Redirection operator `>` creates/overwrites file
- Cat displays entire file contents
- Default permissions: `-rw-r--r--` (644)

---

### Experiment 2: Permission Model and Execute Bit

**Objective**: Demonstrate permission model and the necessity of execute bit for scripts.

**Hypothesis**: Scripts cannot be executed without the execute permission bit set. Adding execute makes them runnable.

**Setup**:

```bash
ls -l                          # View current directory
# Output shows: backup.sh, jenkins.sh, log-collector.sh all as shell scripts
```

**Steps**:

1. Check current permissions: `ls -l backup.sh`
   - Output: `-rw-rw-r-- 1 ubuntu ubuntu 245 Feb 15 05:41 backup.sh`
2. Try executing without permission: `./backup.sh`
   - Error: `Permission denied`
3. Add execute: `chmod +x backup.sh`
4. Verify permission added: `ls -l backup.sh`
   - Output: `-rwxrwxr-x 1 ubuntu ubuntu 245 Feb 15 05:41 backup.sh`
5. Execute script: `./backup.sh`
   - Success (script runs)

**Results**:

```
✓ Confirmed execute bit required to run scripts
✓ chmod +x successfully added execute permission
✓ Permission change verified with ls -l
✓ Script now executable
```

**Permission Breakdown**:

```
-rwxrwxr-x
├─ First character (-) = regular file
├─ rwx (owner) = read + write + execute
├─ rwx (group) = read + write + execute
└─ r-x (others) = read + execute only
```

**Numeric Form**: `775` (7=rwx, 7=rwx, 5=r-x)

**Observations**:

- File type indicator critical (- for file, d for directory)
- Three permission groups: owner, group, others
- Each group has three permissions: r, w, x
- Execute bit is the difference between readable file and executable script

---

### Experiment 3: User Creation and Home Directory Provisioning

**Objective**: Demonstrate proper user creation and the importance of home directory.

**Hypothesis**: Using `useradd` without `-m` flag creates a user but fails to create home directory, breaking the user's login environment.

**Setup**:

```bash
# Attempt 1: Wrong way
sudo useradd devops
su - devops
```

**Results** (Attempt 1 - Wrong):

```
su: warning: cannot change directory to /home/devops
$ (shell prompt appears but broken)
```

**Issue Identified**:

- User created but home directory does not exist
- Shell cannot enter home directory
- User environment incomplete

**Correction Steps**:

```bash
exit                                     # Exit broken shell
sudo userdel devops                      # Delete broken user
sudo useradd -m devops                   # Create with -m flag
sudo passwd devops                       # Set password
su - devops                              # Switch user
```

**Results** (Attempt 2 - Correct):

```
devops@ubuntu:~$                         # Correct prompt
pwd                                      # Shows /home/devops
ls /home/devops                          # Home directory exists
```

**Key Findings**:

- `useradd -m` creates home directory
- Home directory contains shell dotfiles (.bashrc, .profile)
- Without home directory, user environment is broken
- Always verify: `ls /home/username`

**Comparison**:
| Command | Creates Home | Sets Shell | Asks Password | Use For |
|---------|--------------|-----------|---|---------|
| useradd -m | Yes (with -m) | No (optional) | No | Scripting |
| adduser | Yes | Yes (bash) | Yes | Manual |

**Recommendation**: Use `sudo adduser username` for manual user creation (easier, fewer mistakes).

---

### Experiment 4: Sudo Access and Group Membership

**Objective**: Demonstrate how sudo privileges are controlled and how switching users affects privileges.

**Hypothesis**: Sudo access is group-based. Switching to a non-sudo user eliminates sudo capabilities until switching back.

**Setup**:

```bash
# Create user with sudo access
sudo adduser devops
sudo usermod -aG sudo devops
```

**Step 1: Verify sudo access (from ubuntu user)**:

```bash
groups devops
# Output: devops adm sudo
sudo userdel devops                      # Works (ubuntu has sudo)
```

**Step 2: Demonstrate loss of sudo in switched user context**:

```bash
su - devops
sudo userdel devops
# Error: devops is not in the sudoers file
```

**Root Cause**: The user `devops` is in the `sudo` group, but the `sudo` command is being executed in the `devops` user's context from a non-sudo login shell (not from a sudo shell).

**Observation**:

```
devops@ubuntu:~$ whoami
devops

devops@ubuntu:~$ id
uid=1001(devops) gid=1001(devops) groups=1001(devops),27(sudo)

devops@ubuntu:~$ sudo userdel devops
sudo: /etc/sudoers is owned by uid 1000, should be uid 0
```

**Recovery Steps**:

```bash
exit                                     # Back to ubuntu user
sudo usermod -aG sudo devops            # Ensure sudo group membership
su - devops
# Now user should have sudo access
```

**Key Findings**:

- User must be in `sudo` group for sudo access
- Switching users with `su -` changes the security context
- Can always exit back to privileged user to fix permissions
- Verify groups: `groups username`

**Operational Impact**:

- Never switch to non-sudo user to manage system
- Always check group membership before switching
- If locked out, can switch back to admin user

---

### Experiment 5: Package Management and Repository Failures

**Objective**: Demonstrate how stale package cache causes installation failures and recovery process.

**Hypothesis**: Outdated package repository metadata causes 404 errors during installation. Running `apt update` refreshes cache and resolves the issue.

**Setup**:

```bash
sudo apt install nginx -y
```

**Expected**: nginx installs and service is available

**Result** (Failed):

```
E: Unable to fetch some archives
E: 404 Not Found
nginx package installation failed
systemctl status nginx
# Unit nginx.service not found (service doesn't exist)
```

**Root Cause**: Package repository metadata was outdated, causing package lookup failures.

**Debugging Steps**:

```bash
sudo apt update
# Output: Repository metadata refreshed
sudo apt --fix-broken install
# Fixes any incomplete installations
```

**Resolution Steps**:

```bash
sudo apt update
sudo apt --fix-broken install
sudo apt upgrade -y
sudo apt install nginx -y
# Installation succeeds
```

**Verification**:

```bash
dpkg -l | grep nginx
# ii  nginx  (installed successfully)
sudo systemctl start nginx
systemctl status nginx
# active (running)
ss -tulpn | grep :80
# LISTEN 0.0.0.0:80
```

**Key Findings**:

- Package repository metadata becomes stale over time
- Stale metadata causes 404 errors (package not found)
- `apt update` must be run to refresh metadata
- Failed package installations leave broken state
- `apt --fix-broken install` recovers broken state

**Prevention**:

```bash
# Always run update before package operations on Debian/Ubuntu
sudo apt update
sudo apt install package-name -y
```

**Operational Impact**: Most package installation failures can be resolved with `apt update` → `apt --fix-broken install` → `apt install` cycle.

---

### Experiment 6: Kernel Upgrades and System Resource Constraints

**Objective**: Demonstrate how system upgrades trigger kernel updates and how resource constraints manifest.

**Hypothesis**: Heavy system operations (kernel upgrade, package upgrades) on constrained VMs produce "soft lockup" warnings, indicating resource saturation.

**Setup**:

```bash
sudo apt upgrade -y                      # Full system upgrade
```

**Observations During Upgrade**:

```
Processing triggers for initramfs-tools (0.142)
update-initramfs: Generating /boot/initrd.img-6.8.0-117-generic
update-grub: Generating grub configuration file ...

watchdog: BUG: soft lockup - CPU#1 stuck for 130s
[  412.956844] Kernel panic - not syncing: Watchdog detected hard lockup
```

**What This Means**:

- CPU became unresponsive temporarily (watchdog detected it)
- System under extreme load during kernel/initramfs rebuild
- VirtualBox resource constraints (low RAM or CPU cores)
- Not necessarily fatal (system recovered)

**Recovery**:

```bash
sudo reboot                              # Reboot to new kernel
```

**Verification**:

```bash
uname -r
# Before: 6.8.0-101-generic
# After:  6.8.0-117-generic (new kernel)
```

**Key Findings**:

- Kernel upgrades trigger initramfs rebuild (heavy I/O)
- Heavy operations on constrained VMs cause temporary hangs
- "Soft lockup" warning indicates resource saturation
- Reboot activates new kernel
- System usually recovers (hard lockup would be catastrophic)

**Mitigation**:

```bash
# Run system upgrades when can tolerate brief unresponsiveness
# Increase VM resources if frequent soft lockups occur:
# - VirtualBox: Settings → System → Processor (increase CPUs)
# - VirtualBox: Settings → System → Base Memory (increase RAM)
```

**Operational Lesson**: Kernel upgrades are heavy operations. Allocate sufficient VM resources or run during maintenance windows.

---

### Experiment 7: Service Port Binding Conflicts

**Objective**: Demonstrate port binding conflicts and conflict resolution strategies.

**Hypothesis**: Multiple services cannot bind to the same port. Attempting to start a service on an already-bound port fails with "bind() failed" error.

**Setup**:

```bash
sudo apt install apache2                 # Alternative web server (also uses port 80)
sudo systemctl start apache2
# Now port 80 is occupied by apache2

sudo systemctl start nginx
# Attempt to start nginx on port 80
```

**Result** (Conflict):

```
Job for nginx.service failed
systemctl status nginx
# showed error about port binding
journalctl -xeu nginx
# Error: bind() to 0.0.0.0:80 failed
```

**Debugging**:

```bash
sudo ss -tulpn | grep :80
# LISTEN 0 511 0.0.0.0:80 users:(("apache2",pid=1456))
```

**Resolution Strategy 1** (Stop conflicting service):

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2           # Don't auto-start
sudo systemctl start nginx
sudo systemctl status nginx
# active (running)
```

**Resolution Strategy 2** (Change nginx port):

```bash
sudo nano /etc/nginx/sites-available/default
# Change: listen 80; → listen 8080;
sudo systemctl restart nginx
curl http://localhost:8080               # Works on alternate port
```

**Verification**:

```bash
sudo ss -tulpn | grep :80               # Check port 80
sudo ss -tulpn | grep :8080             # Check port 8080
curl localhost:8080                     # Verify service responds
```

**Key Findings**:

- Linux allows only ONE process per port
- Port conflicts are legitimate infrastructure failures
- Error message clearly indicates bind failure
- Can identify conflicting process with `ss` or `lsof`
- Two solutions: remove conflict or change port

**Prevention**:

```bash
# Always check port before starting service
sudo ss -tulpn | grep :80
# If occupied, either:
# 1. Stop the occupying service
# 2. Use different port
```

**Operational Pattern**: Port conflicts are common with web servers (nginx, apache, node apps all default to 80).

---

### Experiment 8: Service Configuration Validation

**Objective**: Demonstrate configuration testing workflow before applying changes.

**Hypothesis**: Invalid configuration syntax will prevent service restart. Testing configuration beforehand prevents broken services.

**Setup**:

```bash
sudo nano /etc/nginx/nginx.conf
# Add a garbage line: broken_config
```

**Step 1: Attempt restart with broken config**:

```bash
sudo systemctl restart nginx
# Job for nginx.service failed
```

**Step 2: Identify problem**:

```bash
systemctl status nginx
# Shows failed but limited detail
journalctl -xeu nginx
# Shows more detail but still cryptic
sudo nginx -t
# Shows EXACT problem: "configuration file test failed" + line number
```

**Step 3: Fix the configuration**:

```bash
sudo nano /etc/nginx/nginx.conf
# Remove garbage line
```

**Step 4: Validate configuration**:

```bash
sudo nginx -t
# Output: syntax is ok
#         test is successful
```

**Step 5: Restart service**:

```bash
sudo systemctl restart nginx
systemctl status nginx
# active (running)
```

**Key Findings**:

- Configuration errors prevent service start
- `systemctl status` shows failure but not specific error
- `journalctl -xeu` shows more context
- `nginx -t` gives specific syntax error with line number
- Must fix config, test, then restart

**Best Practice Workflow**:

```bash
# BEFORE restarting:
sudo nginx -t                            # Validate config

# THEN restart if valid:
sudo systemctl restart nginx

# VERIFY:
systemctl status nginx
curl localhost                           # Test service
```

**Prevention**:

```bash
# Always test before applying
sudo service-name -t                     # If service supports it
# or use dry-run modes
```

---

## Experiment Summary

| #   | Objective            | Key Finding                                     | Operational Impact                              |
| --- | -------------------- | ----------------------------------------------- | ----------------------------------------------- |
| 1   | Basic filesystem ops | Touch creates files with 644 permissions        | Normal file creation works as expected          |
| 2   | Permission model     | Execute bit required for scripts                | Script execution requires chmod +x              |
| 3   | User creation        | Home directory critical for user function       | Always use -m or adduser                        |
| 4   | Sudo and groups      | Privileges are group-based                      | Group membership must be verified               |
| 5   | Package management   | Stale cache causes 404 errors                   | apt update required before installs             |
| 6   | Kernel upgrades      | Heavy ops cause soft lockups on constrained VMs | Allocate resources or run during maintenance    |
| 7   | Port conflicts       | Multiple services cannot share port             | Check port availability before starting service |
| 8   | Config validation    | Invalid config prevents service start           | Always test config before restart               |
