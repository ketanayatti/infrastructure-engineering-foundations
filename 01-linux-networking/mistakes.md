# Mistakes and Pitfalls

Document mistakes to avoid repetition and reduce operational risk.

## User Management Pitfalls

### Mistake 1: Creating User Without Home Directory

**What Happened**:

```bash
sudo useradd devops          # Without -m flag
su - devops
# Error: su: warning: cannot change directory to /home/devops
# Shell is broken, cannot work properly
```

**Root Cause**: `useradd` by default does NOT create home directory unless `-m` flag is used.

**Impact**: User created but non-functional - cannot enter home directory, shell environment broken.

**Correction**:

```bash
sudo userdel devops                      # Remove broken user
sudo useradd -m devops                   # Create with home
# OR (better for manual):
sudo adduser devops                      # Creates home automatically
sudo passwd devops                       # Set password
su - devops                              # Now works
```

**Verification**:

```bash
ls /home/devops                          # Home directory exists
su - devops && pwd                       # Should show /home/devops
```

**Prevention**: Always use `sudo adduser` for manual user creation. Use `sudo useradd -m` only when scripting.

### Mistake 2: Switching User, Losing Sudo Access

**What Happened**:

```bash
su - devops                              # Switch to non-sudo user
sudo userdel devops
# Error: sudo: devops is not in the sudoers file
# Now cannot perform admin operations
```

**Root Cause**: `su - devops` switches to devops user context. Devops user doesn't have sudo access, so sudo commands fail.

**Impact**: Stuck in unprivileged shell context. Cannot run system commands.

**Correction**:

```bash
exit                                     # Exit unprivileged shell
# Back to original user with sudo access
sudo usermod -aG sudo devops            # Add devops to sudo group
su - devops                              # Now can use sudo
```

**Prevention**: Check group membership before switching users:

```bash
groups devops                            # Verify sudo group
```

**Key Lesson**: Switching to non-privileged user with `su -` removes all sudo capabilities. Exit back to privileged user to manage admin operations.

### Mistake 3: Using -G Without -a (Replaces Groups)

**What Happened**:

```bash
sudo usermod -G docker ubuntu            # Without -a
# This REPLACES all groups, not appends
groups ubuntu
# Output: docker
# Lost: ubuntu (primary group), sudo access
```

**Root Cause**: `-G` sets groups (replace). `-a` means append. Without `-a`, all existing groups are lost.

**Impact**: User loses access to critical groups (sudo, primary group). Can break system access.

**Correction**:

```bash
sudo usermod -aG docker ubuntu          # -a means append, keep existing
groups ubuntu
# Output: ubuntu docker sudo
# All groups preserved
```

**Prevention**: Always use `-aG` together:

- `-a` = append (keep existing groups)
- `-G` = set groups (replace)
- ALWAYS use them together: `-aG`

**Pattern**:

```bash
# CORRECT:
sudo usermod -aG docker ubuntu

# WRONG (loses groups):
sudo usermod -G docker ubuntu
```

## Service Management Pitfalls

### Mistake 1: Service Not Found After Installation Failure

**What Happened**:

```bash
sudo apt install nginx -y
# Installation fails with 404 errors
systemctl start nginx
# Error: Unit nginx.service not found
```

**Root Cause**: Package installation failed due to stale apt cache. nginx package never actually installed.

**Why systemctl fails**: Service file doesn't exist if package installation didn't complete.

**Debugging**:

```bash
dpkg -l | grep nginx                    # Check if installed
# Output: (empty) - not installed
journalctl -xe                          # Check installation logs
```

**Correction**:

```bash
sudo apt update                          # Refresh package metadata
sudo apt --fix-broken install            # Fix any broken state
sudo apt upgrade -y                      # Upgrade packages
sudo apt install nginx -y                # Install again
# Verify:
dpkg -l | grep nginx
# Should show: ii nginx
```

**Prevention**: Always run `apt update` before package operations on Debian/Ubuntu systems.

### Mistake 2: Port Already in Use (Conflict)

**What Happened**:

```bash
sudo systemctl start nginx
# Job for nginx.service failed
journalctl -xeu nginx
# Error: bind() to 0.0.0.0:80 failed
```

**Root Cause**: Another process already listening on port 80.

**Debugging Steps**:

```bash
sudo ss -tulpn | grep :80
# Output: apache2 already listening on :80
```

**Correction Option 1** (stop conflicting service):

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo systemctl start nginx
sudo systemctl status nginx
# Should now be active (running)
```

**Correction Option 2** (change nginx port):

```bash
sudo nano /etc/nginx/sites-available/default
# Change: listen 80; → listen 8080;
sudo systemctl restart nginx
curl http://localhost:8080             # Now works on 8080
```

**Prevention**: Check port before starting service:

```bash
sudo ss -tulpn | grep :80               # Already in use?
sudo lsof -i :80                        # Which process?
```

### Mistake 3: Broken Configuration Doesn't Prevent Restart Attempt

**What Happened**:

```bash
sudo nano /etc/nginx/nginx.conf         # Add garbage line
sudo systemctl restart nginx
# Job for nginx.service failed
systemctl status nginx
# Shows failed but no error details
```

**Root Cause**: Config syntax error. Service manager reports failure but not the specific error.

**Debugging**:

```bash
sudo nginx -t                            # Test syntax
# Output: nginx: configuration file test failed
# Shows which file and line number
```

**Correction**:

```bash
sudo nano /etc/nginx/nginx.conf         # Fix the garbage line
sudo nginx -t                            # Verify syntax now valid
# Output: syntax is ok, test is successful
sudo systemctl restart nginx
systemctl status nginx
# Should be active (running)
```

**Prevention**: Always test configuration before restart:

```bash
sudo nginx -t && sudo systemctl restart nginx
```

Prevents applying broken configs.

## Filesystem and Permissions Pitfalls

### Mistake 1: Wrong File Permissions on Executables

**What Happened**:

```bash
touch script.sh                          # Create script
echo "echo hello" > script.sh
./script.sh
# Error: Permission denied
ls -l script.sh
# Output: -rw-r--r-- (no execute bit)
```

**Root Cause**: `touch` creates files with default permissions (644), which doesn't include execute.

**Correction**:

```bash
chmod +x script.sh
./script.sh
# Output: hello
```

**Verification**:

```bash
ls -l script.sh
# Output: -rwxr-xr-x (execute bit added)
```

**Prevention**: Create executable files directly with correct permissions:

```bash
# Option 1: Create then chmod
touch script.sh
chmod +x script.sh

# Option 2: Specify permissions on creation
install -m 755 -c script.sh /usr/local/bin/

# Option 3: Use shell script syntax
cat > script.sh << 'EOF'
#!/bin/bash
echo hello
EOF
chmod +x script.sh
```

### Mistake 2: Wrong Ownership Causes Access Denied

**What Happened**:

```bash
touch /var/www/html/app.log              # Create as ubuntu
# Web server needs to write to it
sudo systemctl start nginx
# nginx running as www-data
# nginx cannot write to file (ubuntu owned)
# Error: Permission denied in web app
```

**Root Cause**: File owned by ubuntu, but service runs as www-data. www-data lacks write permission.

**Verification**:

```bash
ls -l /var/www/html/app.log
# Output: -rw-r--r-- ubuntu ubuntu
ps aux | grep nginx
# Output: www-data is the nginx process owner
```

**Correction**:

```bash
sudo chown www-data:www-data /var/www/html/app.log
ls -l /var/www/html/app.log
# Output: -rw-r--r-- www-data www-data
```

**Prevention**: Assign correct ownership immediately when creating service resources:

```bash
sudo touch /var/www/html/app.log
sudo chown www-data:www-data /var/www/html/app.log
```

Or create with correct ownership:

```bash
sudo install -o www-data -g www-data -m 644 /dev/null /var/www/html/app.log
```

### Mistake 3: Recursive Chmod/Chown on Wrong Directory

**What Happened**:

```bash
sudo chmod 777 -R /                      # Recursive, root directory (CATASTROPHIC)
# Now ENTIRE SYSTEM is world-writable
# Security disaster
```

**Root Cause**: Used recursive flag on system-critical directory without thinking.

**Impact**: Complete system compromise. Everyone can modify everything.

**Prevention**:

- Always specify exact directory (never `/` or `/usr` or `/etc`)
- Use `-R` very carefully
- Test on single file first
- Use find with limits:

```bash
# Bad (never do this):
sudo chmod -R 777 /

# Good (specific directory):
sudo chmod -R 755 /var/www/html

# Better (test first):
chmod 755 test_file                  # Test on one file
sudo chmod -R 755 /var/www/html      # Then recursive
```

## Process Management Pitfalls

### Mistake 1: Force Kill When Graceful Would Work

**What Happened**:

```bash
ps aux | grep python
# Output: ubuntu 1234 ... python app.py
kill -9 1234                             # SIGKILL immediately
# Process killed without cleanup
# Database transaction not rolled back
# Temporary files not cleaned
```

**Root Cause**: Used SIGKILL (-9) instead of SIGTERM (15).

**Impact**: Unclean shutdown can corrupt data or leave orphaned resources.

**Correction Pattern**:

```bash
kill 1234                                # SIGTERM (graceful)
sleep 2
ps aux | grep 1234                       # Still running?
kill -9 1234                             # Only if still running
```

**Prevention**: Always use graceful termination first:

```bash
kill PID              # Graceful (SIGTERM 15)
# Wait a reasonable time
kill -9 PID           # Force (SIGKILL 9) only if graceful fails
```

### Mistake 2: Creating Background Process, Forgetting to Monitor

**What Happened**:

```bash
sleep 300 &                              # Start background job
# Prompt returns, shell continues
# Later: system slow, many sleep processes running
ps aux | grep sleep
# Output: 47 sleep processes
```

**Root Cause**: Background processes keep running. Easy to forget them and create duplicates.

**Cleanup**:

```bash
jobs                                     # List current shell's jobs
kill %1                                  # Kill job by number
ps aux | grep sleep | awk '{print $2}' | xargs kill   # Kill all
```

**Prevention**: Kill background jobs when done:

```bash
sleep 300 &
# Do work...
kill %1                                  # Clean up before exiting
```

## Debugging Workflow Failures

### Mistake 1: Not Checking Basic System State First

**What Happened**:

```bash
"Service won't start"
# Immediately tries complex debugging
# Doesn't check:
whoami                                   # Am I running with sudo?
pwd                                      # Am I in right directory?
groups                                   # Do I have needed groups?
```

**Correction**: Always start with basic checks:

```bash
# 1. Identity check
whoami                                   # ubuntu or root?
id                                       # Full group info
pwd                                      # Current location

# 2. Service check
systemctl status nginx                   # Status?
journalctl -xeu nginx                    # Detailed error?

# 3. Port check
sudo ss -tulpn | grep :80               # Port conflict?

# 4. File check
ls -l /etc/nginx                         # Permissions?
sudo nginx -t                            # Config valid?
```

### Mistake 2: Not Reading Full Error Message

**What Happened**:

```bash
sudo systemctl restart nginx
# Error (partial): Job for nginx.service failed
# Assumption: nginx is broken
# Truth: Config syntax error on line 42
```

**Correction**: Read the complete error:

```bash
journalctl -xeu nginx                    # Full context
sudo nginx -t                            # Specific line with error
systemctl status nginx                   # Expanded output
```

**Prevention**: Use diagnostic tools that give full context:

```bash
systemctl status servicename             # Shows much more than just "failed"
journalctl -xeu servicename              # Extremely detailed
```

## Summary of Common Patterns

| Issue               | First Check                              | Fix                                             |
| ------------------- | ---------------------------------------- | ----------------------------------------------- |
| Permission denied   | `ls -l filename`                         | `sudo chown user:group file` or `chmod +x file` |
| Port conflict       | `sudo ss -tulpn \| grep :PORT`           | Stop conflicting service or change port         |
| Service won't start | `systemctl status` and `journalctl -xeu` | Usually config syntax or dependency issue       |
| User can't sudo     | `groups username`                        | `sudo usermod -aG sudo username`                |
| Process won't stop  | `kill PID` then `kill -9 PID`            | Graceful first, force if needed                 |
| Package not found   | `sudo apt update`                        | Refresh cache before installing                 |
| No home directory   | `ls /home/username`                      | `sudo useradd -m` or `sudo adduser`             |
