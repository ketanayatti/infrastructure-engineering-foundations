# Learnings

Key learnings captured over time.

## Day 1 Session: Fundamentals through System Services

### Core Operational Insights

#### 1. Permissions Are the First Debugging Layer

**Discovery**: When troubleshooting, 40%+ of failures stem from permission or ownership issues, not from the application itself.

**Pattern**: Most deployment failures follow this order:

1. Wrong permissions (file not readable/executable)
2. Wrong ownership (service user cannot access resources)
3. Wrong user/group membership (service lacks required privileges)
4. Application-level issues (actual code/config problems)

**Operational Impact**: Always run this check first:

```bash
ls -l filename                    # Check file permissions
ps aux | grep service_name        # Check service user
whoami                            # Verify current user
```

Saves hours of debugging when checked first.

#### 2. Permission Inheritance Model

**Discovery**: Permissions on parent directory control access to children.

**Model**:

- To LIST directory contents: `r` on directory
- To ENTER directory: `x` on directory
- To READ file: `r` on file
- To WRITE file: `w` on file
- To EXECUTE file: `x` on file

**Critical**: A file with `rw-------` (600) inside a directory with `r--r--r--` (644) is still readable by others because directory is readable.

**Operational**: Always verify both directory and file permissions:

```bash
ls -l /path/to/          # Directory permissions
ls -l /path/to/file      # File permissions
```

#### 3. User Context Execution Determines Behavior

**Discovery**: What a process can do depends entirely on which user runs it.

**Examples**:

- nginx running as www-data can only access www-data-owned files
- docker running as root can access all files
- Script owned by ubuntu with 755 permissions runs with ubuntu's privileges

**Operational Implication**: Service permission denied errors are almost always:

```bash
sudo chown -R service_user:service_user /resource/path
```

Fixes 90% of "permission denied" issues.

#### 4. useradd vs adduser: Scripting vs Interactive

**Discovery**: Different commands for different purposes.

| Tool         | Created Home?     | Sets Shell?   | Creates Group? | Use For               |
| ------------ | ----------------- | ------------- | -------------- | --------------------- |
| `useradd -m` | Only with -m flag | No            | No             | Scripting, automation |
| `adduser`    | Always            | Always (bash) | Always         | Manual admin work     |

**Learned**: Using raw `useradd` without `-m` creates broken home directory scenarios.

**Standard Practice**:

```bash
sudo adduser username              # Human-friendly
sudo useradd -m username           # Scripting-friendly
```

#### 5. sudo Access is Group-Based

**Discovery**: sudo privileges are controlled through group membership, not individual permission bits.

**Model**:

- User in `sudo` group → `sudo` access
- User not in `sudo` group → no `sudo`

**Operational Issue**: When you switch to a non-sudo user (`su - devops`), you lose sudo privileges immediately. No way to recover without switching back to a privileged user.

**Fix Pattern**:

```bash
sudo usermod -aG sudo ubuntu      # Add user to sudo group
# Verify:
groups ubuntu
# Should show: ubuntu adm sudo docker
```

#### 6. Process Lifecycle and Signal Handling

**Discovery**: Not all process termination is equal.

| Signal       | Behavior                                 | When to Use           |
| ------------ | ---------------------------------------- | --------------------- |
| SIGTERM (15) | Graceful shutdown - process can clean up | Always try first      |
| SIGKILL (9)  | Forced termination - no cleanup possible | Only if SIGTERM fails |

**Operational Pattern**:

```bash
kill PID              # SIGTERM (15) - graceful
sleep 5
ps aux | grep PID     # Check if gone
kill -9 PID           # Only if still running
```

**Learning**: Scripting tools often use SIGKILL prematurely, leaving orphaned resources or incomplete transactions.

#### 7. systemd Auto-Restart Masks Root Causes

**Discovery**: Service auto-restart (`Restart=always`) improves availability but hides problems.

**Observation**:

```bash
systemctl status nginx
# Shows: active (running)
# But journalctl shows: Service restarted 47 times in 1 hour
```

**Operational Implication**: High restart count = serious underlying issue that needs investigation. Don't ignore it.

**Debugging Workflow**: When you see frequent restarts:

1. Check latest logs: `journalctl -xeu nginx -n 50`
2. Identify failure pattern
3. Fix root cause
4. Monitor restart frequency afterward

#### 8. journalctl Preserves Context (vs tail -f)

**Discovery**: journalctl maintains structure and queryability; tail -f loses it.

**Example Comparison**:

With `tail -f /var/log/nginx/error.log`:

```
2024-05-25 10:00:01 connect() failed
2024-05-25 10:00:02 permission denied
(context limited to text)
```

With `journalctl -u nginx`:

```
● nginx.service - nginx HTTP Server
   Loaded: loaded
   Active: active (running)
   Service: nginx
   UID/GID: www-data
   Status: "Binding to port 80..."
   Since: 2024-05-25 09:55:00
```

**Operational Advantage**: journalctl shows service metadata, dependencies, and structured context.

#### 9. Port Binding is a Resource Contention Point

**Discovery**: Multiple services cannot share the same port. This is a real, common failure.

**Observed Scenario**:

```bash
sudo systemctl start nginx
# Error: Job for nginx.service failed
journalctl -xeu nginx
# "bind() to 0.0.0.0:80 failed"
```

**Debugging**:

```bash
sudo ss -tulpn | grep :80
# Shows: apache2 already listening on port 80
```

**Fix**: Stop conflicting service and start nginx, or change nginx port.

**Operational Lesson**: Port conflicts are legitimate infrastructure problems, not "wrong configuration."

#### 10. Package Manager Cache Staleness Causes Silent Failures

**Discovery**: Old apt cache can cause installations to fail silently.

**Observed Scenario**:

```bash
sudo apt install nginx
# Error: 404 Not Found
# nginx package never installs
systemctl status nginx
# Unit nginx.service not found
```

**Root Cause**: Package repository metadata was outdated, so package lookup failed.

**Fix**:

```bash
sudo apt update          # Refresh package metadata
sudo apt --fix-broken install
sudo apt upgrade -y
sudo apt install nginx -y
```

**Operational Pattern**: Always run `apt update` before troubleshooting package issues on Debian/Ubuntu.

#### 11. VM Resource Constraints Appear as System Hangs

**Discovery**: Low VM resources (RAM, CPU) manifest as "soft lockup" warnings during heavy operations.

**Observed**:

```
watchdog: BUG: soft lockup - CPU#1 stuck for 130s
```

**Means**: System under heavy load, unresponsive temporarily.

**Causes**:

- Kernel upgrade (heavy I/O)
- Package upgrade (CPU-intensive)
- VirtualBox resource limits (not enough RAM/CPU)

**Mitigation**:

```bash
sudo reboot                                # After upgrade
# Verify new kernel:
uname -r                                   # Should show new version
```

#### 12. Authentication vs Authorization Are Different

**Discovery**: Two separate concerns:

| Aspect         | Mechanism                    | Failure Type                                |
| -------------- | ---------------------------- | ------------------------------------------- |
| Authentication | User login/password          | "User doesn't exist" or "Permission denied" |
| Authorization  | Group membership/permissions | File permission denied, access denied       |

**Example**:

- `su - ubuntu` fails: Authentication issue (password wrong or user doesn't exist)
- Service can't read file: Authorization issue (wrong ownership/permissions)

**Debugging**: Always check both:

```bash
whoami                    # Authentication verified
ls -l /resource          # Authorization check
id                        # Groups verification
```

## Tradeoffs and Constraints

### Permissions vs Convenience

**Tradeoff**: Strict permissions (600) are secure but inconvenient; loose permissions (777) are convenient but dangerous.

**Industry Standard**: Use `755` for executables, `644` for files, `600` for secrets.

### Service Auto-Restart vs Stability

**Tradeoff**: Auto-restart improves uptime but hides failures. Monitor restart frequency.

### Sudo Group Membership vs Security

**Tradeoff**: Sudo access is powerful. Docker group membership is equivalent to sudo (can escape to root).

**Operational Note**: Don't add users to docker group carelessly. It's a privilege escalation vector.

### systemd Abstraction vs Direct Service Management

**Advantage**: systemd provides consistent interface, dependency management, auto-restart.

**Disadvantage**: Lower-level debugging harder (can't directly see process internals).

## Follow-ups

- [ ] Investigate Linux capabilities as alternative to SUID bits
- [ ] Test socket activation vs traditional service startup performance
- [ ] Research SELinux/AppArmor permission models
- [ ] Study systemd hardening best practices
- [ ] Experiment with custom service files for infrastructure automation
- [ ] Compare journalctl query performance on large logs
- [ ] Test signal handling in different programming languages (Go, Python, Rust)
