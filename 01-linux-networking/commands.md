# Commands

Curated commands for Linux and networking operations. Add context, expected signal, and safety notes.

## Filesystem Navigation and Inspection

| Command               | Purpose                                  | Expected Output                              | Safety                         |
| --------------------- | ---------------------------------------- | -------------------------------------------- | ------------------------------ |
| `pwd`                 | Print working directory                  | `/home/user/path`                            | Safe                           |
| `ls`                  | List directory contents                  | Files and folders                            | Safe                           |
| `ls -l`               | Long format (with permissions)           | `-rw-r--r-- 1 user group 245 May 25 app.log` | Safe                           |
| `ls -la`              | Include hidden files (starting with `.`) | Includes dotfiles                            | Safe                           |
| `cd /path`            | Change directory                         | Changes prompt path                          | Safe                           |
| `mkdir dirname`       | Create directory                         | Directory created                            | Safe                           |
| `touch filename`      | Create empty file                        | Empty file created                           | Safe                           |
| `cp source dest`      | Copy file                                | Destination contains copy                    | Safe                           |
| `mv source dest`      | Move/rename file                         | File at new location                         | Safe                           |
| `rm filename`         | Remove file                              | File deleted permanently                     | ⚠️ Permanent, use with care    |
| `cat filename`        | Display file contents                    | Full file output                             | Safe                           |
| `less filename`       | Browse file (page by page)               | Paginated view, press `q` to exit            | Safe                           |
| `echo "text"`         | Print text                               | Text output                                  | Safe                           |
| `echo "text" > file`  | Redirect to file (overwrite)             | File overwritten with text                   | ⚠️ Overwrites existing content |
| `echo "text" >> file` | Append to file                           | Text added to end                            | Safe                           |

## File Permissions

### Viewing Permissions

| Command          | Purpose                    | Expected Output                              | Notes                                               |
| ---------------- | -------------------------- | -------------------------------------------- | --------------------------------------------------- |
| `ls -l filename` | Show file with permissions | `-rw-r--r-- 1 user group 245 May 25 app.log` | First part shows permissions, ownership, size, date |
| `stat filename`  | Detailed file information  | Permission bits, UID, GID, timestamps        | More verbose than ls                                |

### Changing Permissions

| Command               | Purpose                     | Example                 | Effect                                | Safety                           |
| --------------------- | --------------------------- | ----------------------- | ------------------------------------- | -------------------------------- |
| `chmod +x script.sh`  | Add execute for all         | `chmod +x script.sh`    | Makes file executable                 | Safe                             |
| `chmod -x script.sh`  | Remove execute for all      | `chmod -x script.sh`    | Makes file not executable             | Safe                             |
| `chmod 755 script.sh` | Set to rwxr-xr-x            | `chmod 755 script.sh`   | Owner full, group/others read+execute | Safe                             |
| `chmod 644 file`      | Set to rw-r--r--            | `chmod 644 file`        | Owner read+write, others read-only    | Safe                             |
| `chmod 600 secret`    | Set to rw-------            | `chmod 600 secret`      | Owner only                            | Safe                             |
| `chmod 777 file`      | Set to rwxrwxrwx            | `chmod 777 file`        | Everyone full access                  | ⚠️ Never in production           |
| `chmod -R 755 dir/`   | Recursive on directory tree | `chmod -R 755 project/` | Changes all files in directory        | ⚠️ Can change many files at once |

### Changing Ownership

| Command                         | Purpose                    | Example                                | Effect                       | Safety                               |
| ------------------------------- | -------------------------- | -------------------------------------- | ---------------------------- | ------------------------------------ |
| `chown user file`               | Change file owner          | `chown ubuntu file.txt`                | Changes owner to ubuntu      | Requires sudo                        |
| `chown user:group file`         | Change owner and group     | `chown ubuntu:docker file.txt`         | Changes both                 | Requires sudo                        |
| `chown -R user dir/`            | Recursive ownership change | `chown -R www-data:www-data /var/www/` | Changes entire tree          | ⚠️ Requires sudo, affects many files |
| `sudo chown -R user:group dir/` | Recursive with sudo        | `sudo chown -R nginx:nginx /app/`      | Safe way to change ownership | Standard practice                    |

**Common Pattern**:

```bash
sudo chown -R www-data:www-data /var/www/html    # Web server files
sudo chown -R mysql:mysql /var/lib/mysql         # Database files
sudo chown -R ubuntu:ubuntu project/              # User project
```

## User and Group Management

### User Information

| Command           | Purpose                     | Expected Output                                                              | Notes                                      |
| ----------------- | --------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------ |
| `whoami`          | Current logged-in user      | `ubuntu`                                                                     | Returns username                           |
| `id`              | Current user and group info | `uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),27(sudo),999(docker)` | Shows UID, GID, and all groups             |
| `groups`          | Current user's groups       | `ubuntu sudo docker`                                                         | Lists all group memberships                |
| `groups username` | Groups for specific user    | `groups ubuntu`                                                              | Shows user's group memberships             |
| `cat /etc/passwd` | All users on system         | `ubuntu:x:1000:1000::/home/ubuntu:/bin/bash`                                 | System user database (readable)            |
| `cat /etc/shadow` | Password hashes             | `ubuntu:$6$...hash...::0:99999:7:::/bin/bash`                                | ⚠️ Requires sudo, contains password hashes |
| `cat /etc/group`  | All groups on system        | `sudo:x:27:ubuntu,ubuntu2`                                                   | Shows groups and members                   |

### User Creation and Modification

| Command                    | Purpose                   | Example                  | Safety                         | Notes                                       |
| -------------------------- | ------------------------- | ------------------------ | ------------------------------ | ------------------------------------------- |
| `sudo adduser username`    | Create user (recommended) | `sudo adduser devops`    | Safe, interactive              | Creates home dir, sets shell, asks password |
| `sudo useradd -m username` | Create user (scripting)   | `sudo useradd -m deploy` | Safe, non-interactive          | Requires -m for home directory              |
| `sudo passwd username`     | Set/change password       | `sudo passwd ubuntu`     | Safe, interactive              | Can reset forgotten passwords               |
| `sudo userdel username`    | Delete user               | `sudo userdel devops`    | ⚠️ Irreversible                | Use -r to also remove home directory        |
| `sudo userdel -r username` | Delete user and home      | `sudo userdel -r devops` | ⚠️ Irreversible, removes files | Complete removal                            |

### Group Management

| Command                           | Purpose                      | Example                              | Safety                 | Notes                                          |
| --------------------------------- | ---------------------------- | ------------------------------------ | ---------------------- | ---------------------------------------------- |
| `sudo groupadd groupname`         | Create group                 | `sudo groupadd docker`               | Safe                   | Group created but empty                        |
| `sudo usermod -aG groupname user` | Add user to group            | `sudo usermod -aG docker ubuntu`     | Safe                   | Append to groups (-a keeps existing)           |
| `sudo usermod -G groupname user`  | Set user's groups (replaces) | `sudo usermod -G docker,sudo ubuntu` | ⚠️ Replaces all groups | Use -a (append) unless replacing intentionally |
| `sudo groupdel groupname`         | Delete group                 | `sudo groupdel tempgroup`            | Safe (if not in use)   | Fails if group has users                       |

**Important Pattern**: Always use `-aG` to append to groups, not replace them. Replacing can accidentally remove sudo access.

### Privilege Escalation

| Command         | Purpose              | Example           | Safety                   | Notes                  |
| --------------- | -------------------- | ----------------- | ------------------------ | ---------------------- |
| `sudo command`  | Run command as root  | `sudo apt update` | Requires sudo access     | One-time escalation    |
| `sudo su`       | Switch to root shell | `sudo su`         | ⚠️ Root shell            | Exit with `exit`       |
| `su - username` | Switch to user       | `su - devops`     | Requires user's password | Full login environment |
| `sudo -i`       | Login shell as root  | `sudo -i`         | ⚠️ Root shell            | Similar to `sudo su`   |

## Process Management

### Viewing Processes

| Command               | Purpose                         | Expected Output                                          | Notes                                |
| --------------------- | ------------------------------- | -------------------------------------------------------- | ------------------------------------ |
| `ps aux`              | All processes full detail       | `USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND` | Most comprehensive view              |
| `ps aux \| grep bash` | Filter processes                | `ubuntu 1234 0.0 0.2 8960 4096 pts/0 S+ 10:00 0:00 bash` | Find specific processes              |
| `top`                 | Real-time process monitor       | Interactive dashboard                                    | Press `q` to quit                    |
| `htop`                | Enhanced process monitor        | Interactive colored dashboard                            | Install with `sudo apt install htop` |
| `pstree`              | Process family tree             | Shows parent-child relationships                         | Visualize process hierarchy          |
| `pgrep bash`          | Find process by name (PID only) | `1234`                                                   | Returns PID(s) matching pattern      |
| `pidof bash`          | Process ID of program           | `1234`                                                   | Alternative to pgrep                 |

### Process Control

| Command        | Purpose                       | Example                  | Signal       | Safety                      |
| -------------- | ----------------------------- | ------------------------ | ------------ | --------------------------- |
| `sleep 300 &`  | Start process in background   | `sleep 300 &`            | -            | Process runs, shell returns |
| `jobs`         | List background jobs          | `[1]+ Running sleep 300` | -            | Shows current shell's jobs  |
| `kill PID`     | Terminate process (graceful)  | `kill 1234`              | SIGTERM (15) | Allows cleanup              |
| `kill -9 PID`  | Force kill process            | `kill -9 1234`           | SIGKILL (9)  | ⚠️ No cleanup, last resort  |
| `kill -15 PID` | Graceful terminate (explicit) | `kill -15 1234`          | SIGTERM (15) | Same as `kill PID`          |
| `kill %1`      | Terminate job by job ID       | `kill %1`                | SIGTERM (15) | Kill by job number, not PID |
| `Ctrl+C`       | Interrupt foreground process  | (interactive)            | SIGINT (2)   | Stops running command       |

**Operational Pattern**: Always try graceful termination first. Use force kill only if graceful fails after waiting.

## System Services (systemctl)

### Service Status

| Command                                    | Purpose                   | Expected Output                                                | Safety               |
| ------------------------------------------ | ------------------------- | -------------------------------------------------------------- | -------------------- |
| `systemctl status nginx`                   | Service status            | `● nginx.service - nginx HTTP Server Active: active (running)` | Safe, read-only      |
| `systemctl is-active nginx`                | Just status (on/off)      | `active` or `inactive`                                         | Safe, minimal output |
| `systemctl is-enabled nginx`               | Boot auto-start status    | `enabled` or `disabled`                                        | Safe, read-only      |
| `systemctl list-units --type=service`      | All services on system    | Long list of all services                                      | Safe, read-only      |
| `systemctl list-unit-files --type=service` | Services with boot config | Shows all service files                                        | Safe, read-only      |

### Service Lifecycle

| Command                        | Purpose                | Example                        | Requires Sudo | Effect                      |
| ------------------------------ | ---------------------- | ------------------------------ | ------------- | --------------------------- |
| `sudo systemctl start nginx`   | Start service now      | `sudo systemctl start nginx`   | Yes           | Service starts immediately  |
| `sudo systemctl stop nginx`    | Stop service           | `sudo systemctl stop nginx`    | Yes           | Service stops               |
| `sudo systemctl restart nginx` | Full restart           | `sudo systemctl restart nginx` | Yes           | Stop + start (full reload)  |
| `sudo systemctl reload nginx`  | Reload config only     | `sudo systemctl reload nginx`  | Yes           | Reload without downtime     |
| `sudo systemctl enable nginx`  | Auto-start on boot     | `sudo systemctl enable nginx`  | Yes           | Adds to boot sequence       |
| `sudo systemctl disable nginx` | Don't auto-start       | `sudo systemctl disable nginx` | Yes           | Removes from boot sequence  |
| `sudo systemctl daemon-reload` | Reload systemd configs | `sudo systemctl daemon-reload` | Yes           | After editing service files |

**Common Workflow**:

```bash
sudo systemctl restart nginx          # After config change
systemctl status nginx                # Verify status
journalctl -xeu nginx                 # Check for errors
curl localhost                        # Test service
```

### Service Logs

| Command                                     | Purpose                   | Example                                     | Output                             |
| ------------------------------------------- | ------------------------- | ------------------------------------------- | ---------------------------------- |
| `journalctl -u nginx`                       | View all logs for service | `journalctl -u nginx`                       | Full log history                   |
| `journalctl -fu nginx`                      | Follow logs live          | `journalctl -fu nginx`                      | Real-time streaming (like tail -f) |
| `journalctl -u nginx -n 20`                 | Last 20 log entries       | `journalctl -u nginx -n 20`                 | Recent logs only                   |
| `journalctl -u nginx -n 50 \| tail -20`     | Last 20 of 50 entries     | Combine for control                         | Filtered view                      |
| `journalctl -u nginx --since "2 hours ago"` | Logs from timeframe       | `journalctl -u nginx --since "2 hours ago"` | Time-filtered logs                 |
| `journalctl -xeu nginx`                     | Detailed failure info     | `journalctl -xeu nginx`                     | Extensive error context            |
| `journalctl -b`                             | All logs since boot       | `journalctl -b`                             | Boot sequence logs                 |
| `journalctl -b -1`                          | Logs from previous boot   | `journalctl -b -1`                          | Previous boot logs                 |

**Most Useful Debugging Command**:

```bash
journalctl -xeu nginx    # Shows detailed error context
```

## Port and Network Inspection

| Command                 | Purpose                       | Example Output                                        | Safety                   |
| ----------------------- | ----------------------------- | ----------------------------------------------------- | ------------------------ |
| `ss -tulpn`             | All listening ports           | `LISTEN 0 511 0.0.0.0:80 users:(("nginx",pid=1234))`  | Safe, read-only          |
| `ss -tulpn \| grep :80` | Specific port                 | `LISTEN ... 0.0.0.0:80 ... nginx`                     | Safe, filter output      |
| `sudo lsof -i :80`      | Process using port            | `nginx 1234 root 6u IPv4 0x123 0t0 TCP *:80 (LISTEN)` | Safe, detailed           |
| `netstat -tulpn`        | Listening ports (older style) | Similar to ss output                                  | Older tool, ss preferred |
| `curl localhost`        | Test HTTP service             | HTML response or error                                | Safe, test connectivity  |
| `curl -I localhost`     | HTTP headers only             | `HTTP/1.1 200 OK`                                     | Safe, minimal output     |

## Configuration Testing

| Command                        | Purpose                   | Example                                                   | Output Meaning            |
| ------------------------------ | ------------------------- | --------------------------------------------------------- | ------------------------- |
| `nginx -t`                     | Test nginx config syntax  | `nginx: configuration file test failed` or `syntax is ok` | Validates before applying |
| `nginx -T`                     | Full config with includes | Shows merged configuration                                | Debugging complex configs |
| `systemctl status servicename` | Full service diagnosis    | Shows config, PID, status, recent logs                    | First debugging step      |

## Quick Diagnostic Workflow

```bash
# Service won't start?
systemctl status nginx
journalctl -xeu nginx
nginx -t

# Port conflict?
sudo ss -tulpn | grep :80
sudo lsof -i :80

# Permission problem?
ls -l /var/www/html
ps aux | grep nginx
cat /etc/passwd | grep www-data

# Which user are you?
whoami
id
pwd
```

## Notes

- **Environment**: Linux (Ubuntu/Debian family assumed)
- **Permissions**: Most system operations require sudo
- **Safety**: Always test changes in non-production first
- **Debugging**: Start with `systemctl status` and `journalctl` for any service issue
- **Signals**: SIGTERM (15) preferred over SIGKILL (9) - allows graceful shutdown
