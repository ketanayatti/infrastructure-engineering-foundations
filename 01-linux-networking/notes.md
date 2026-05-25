# Notes

Living operational notes for Linux and networking. Keep entries concise, evidence-based, and tied to observable behavior.

## Linux System Architecture

### Core Components

**Linux = Kernel + GNU Tools + Shell + Services**

| Component    | Role                             | Operational Relevance               |
| ------------ | -------------------------------- | ----------------------------------- |
| Kernel       | Hardware/process/memory control  | Resource allocation, stability      |
| Shell (bash) | Command interpreter              | User interaction, scripting         |
| Filesystem   | Everything is a file abstraction | Access control, data organization   |
| Process      | Running program instance         | Resource consumption, lifecycle     |
| Service      | Background managed process       | Continuous operation, auto-recovery |

### Standard Filesystem Hierarchy

| Path    | Purpose                         | Contains                               |
| ------- | ------------------------------- | -------------------------------------- |
| `/`     | Root directory                  | All other paths                        |
| `/home` | User files and home directories | User data, dotfiles                    |
| `/etc`  | Configuration files             | System configs, service configs        |
| `/var`  | Variable data, logs             | Logs, temporary data, state            |
| `/tmp`  | Temporary files                 | Session-based temporary data           |
| `/bin`  | Basic system commands           | Essential executables                  |
| `/usr`  | User software                   | Installed applications, libraries      |
| `/proc` | Process information             | Runtime process/kernel state (virtual) |
| `/dev`  | Device files                    | Hardware device interfaces             |

**Operational Observation**: Most deployment failures stem from incorrect file location assumptions or path permissions.

## File Permissions and Ownership Model

### Permission Structure

Every file has three permission groups with three permission types each:

```
-rwxrwxr-x
│││││││││
│││││││└─ Others: execute
│││││││
││││││└── Others: write
│││││
│││││└─── Others: read
│││
││└────── Group: execute
│└─────── Group: write
└──────── Group: read
└────────── First: Owner execute
             Second: Owner write
             Third: Owner read
```

### Permission Types

| Symbol      | Numeric | Meaning            | For Files         | For Directories                  |
| ----------- | ------- | ------------------ | ----------------- | -------------------------------- |
| r (read)    | 4       | Read permission    | Read file content | List directory contents          |
| w (write)   | 2       | Write permission   | Modify file       | Create/delete files in directory |
| x (execute) | 1       | Execute permission | Run as program    | Enter directory                  |

### Permission Groups (In Order)

1. **Owner** (User): File creator or owner
2. **Group**: Members of file's group
3. **Others**: Everyone else

### File Type Indicators

| Symbol | Type          | Example                     |
| ------ | ------------- | --------------------------- |
| `-`    | Regular file  | `-rw-r--r-- script.sh`      |
| `d`    | Directory     | `drwxrwxr-x config/`        |
| `l`    | Symbolic link | `lrwxrwxrwx link -> target` |

### Permission Notation

**Symbolic**: `rwxrwxr-x` (read-write-execute for owner, read-write-execute for group, read-execute for others)

**Numeric**: Convert each group to sum:

- `r = 4`, `w = 2`, `x = 1`
- `rwx = 7` (4+2+1)
- `rw- = 6` (4+2+0)
- `r-x = 5` (4+0+1)

Example: `rwxrwxr-x` = `775`

### Common Permission Patterns

| Permission  | Numeric | Use Case                     | Meaning                               |
| ----------- | ------- | ---------------------------- | ------------------------------------- |
| `rwxr-xr-x` | 755     | Executable scripts, binaries | Owner full access, others can execute |
| `rw-r--r--` | 644     | Regular files, configs       | Owner read/write, others read-only    |
| `rw-------` | 600     | Private files, secrets       | Owner only access                     |
| `rwxrwxrwx` | 777     | Never in production          | Everyone has full access (dangerous)  |

**Operational Note**: Permission issues cause 40%+ of deployment failures. Always verify with `ls -l` before troubleshooting other layers.

## User and Group Management

### User Lifecycle

**User is**: Linux authentication unit with UID, GID, home directory, shell, password.

### Key User Files

| File           | Purpose                                     | Viewable With          |
| -------------- | ------------------------------------------- | ---------------------- |
| `/etc/passwd`  | User account information                    | `cat /etc/passwd`      |
| `/etc/shadow`  | Password hashes (root only)                 | `sudo cat /etc/shadow` |
| `/etc/group`   | Group definitions                           | `cat /etc/group`       |
| `/etc/sudoers` | Sudo permissions (safe editing with visudo) | `sudo visudo`          |

### User Information Structure

From `/etc/passwd`:

```
username:x:1000:1000:Real Name:/home/username:/bin/bash
│        │ │    │    │          │                  │
│        │ │    │    │          │                  └─ Shell
│        │ │    │    │          └─ Home directory
│        │ │    │    └─ GECOS (user comment)
│        │ │    └─ GID (group ID)
│        │ └─ UID (user ID)
│        └─ Password placeholder (actual in /etc/shadow)
└─ Username
```

### Root and Service Users

| User          | UID            | Purpose              | Privilege Level           |
| ------------- | -------------- | -------------------- | ------------------------- |
| root          | 0              | System administrator | Full system access        |
| ubuntu/centos | 1000+          | Human admin account  | Sudo access typically     |
| nginx         | 1000+ (varies) | Web server process   | Limited to web files      |
| mysql         | 1000+ (varies) | Database process     | Limited to database files |

**Operational Pattern**: Services run with dedicated unprivileged users for security isolation.

### Group Membership Impact

User can belong to multiple groups. Group membership grants access to group-owned resources.

**Common DevOps Groups**:

- `sudo`: System administration access
- `docker`: Docker daemon access (security implications - equivalent to sudo)
- `wheel`: Alternative admin group (some distributions)

## Process Model

### Process Lifecycle

```
Created (fork)
    ↓
Running (executing instructions)
    ↓
Blocked (waiting on I/O, lock)
    ↓
Stopped/Killed (exit or signal)
```

### Process States

| State    | Meaning                         | Observable As                            |
| -------- | ------------------------------- | ---------------------------------------- |
| Running  | CPU executing                   | `R` in ps output                         |
| Sleeping | Waiting for event               | `S` in ps output                         |
| Stopped  | Suspended (SIGSTOP)             | `T` in ps output                         |
| Zombie   | Exited, awaiting parent cleanup | `Z` in ps output (indicates parent leak) |

### Process Identification

Every process has:

- **PID** (Process ID): Unique numeric identifier within kernel
- **PPID** (Parent PID): Process that created this process
- **UID**: User who created/owns the process
- **GID**: Group associated with process

### Signal Handling

| Signal  | Number | Behavior                           | Use Case                 |
| ------- | ------ | ---------------------------------- | ------------------------ |
| SIGTERM | 15     | Graceful termination (catchable)   | Standard stop            |
| SIGKILL | 9      | Forced termination (not catchable) | Last resort, force stop  |
| SIGINT  | 2      | Interrupt (Ctrl+C)                 | Interactive cancellation |
| SIGHUP  | 1      | Hangup/reload                      | Service reload           |

**Operational Pattern**: Always use SIGTERM (15) first. Use SIGKILL (9) only when SIGTERM fails after timeout.

## System Services and systemd

### systemd Architecture

**systemd** = System and Service Manager running as PID 1

- Starts and manages all system services
- Handles service dependencies
- Auto-restarts failed services (if configured)
- Manages service logging through journald
- Provides unified service interface via systemctl

### Service Lifecycle

```
[Disabled, inactive]
        ↓
   systemctl enable   (add to boot sequence)
        ↓
   systemctl start    (start now)
        ↓
   [Active running]
        ↓
   systemctl stop     (stop now)
        ↓
   [Inactive]
```

### Service States

| State              | Meaning                            |
| ------------------ | ---------------------------------- |
| `active (running)` | Service is operational             |
| `inactive (dead)`  | Service is stopped                 |
| `failed`           | Service crashed or failed to start |
| `activating`       | Service starting up                |
| `deactivating`     | Service shutting down              |

### Service Configuration

Location: `/etc/systemd/system/` or `/lib/systemd/system/`

Minimal service file structure:

```ini
[Unit]
Description=Service Description

[Service]
ExecStart=/path/to/executable
Restart=always

[Install]
WantedBy=multi-user.target
```

### Service Behavior

| Behavior                | Impact                                | Configuration           |
| ----------------------- | ------------------------------------- | ----------------------- |
| Auto-restart on failure | Improves availability                 | `Restart=always`        |
| Start on boot           | Service runs after system startup     | `systemctl enable`      |
| Dependency ordering     | Ensures service prerequisites are met | `Requires=` or `After=` |

**Operational Insight**: Service auto-restart masks underlying failures. Good for availability, but debug the root cause.

## Logging with journalctl

### Journal Purpose

**journalctl** = Query systemd journal (service logs)

Logs from:

- All systemd-managed services
- Kernel messages
- Early boot messages

### Journal Storage

Binary format (efficient, queryable) stored in:

- `/run/log/journal/` (volatile, survives reboot)
- `/var/log/journal/` (persistent)

### Service vs Traditional Logs

| Aspect           | journalctl                  | Traditional (tail -f) |
| ---------------- | --------------------------- | --------------------- |
| Storage          | Binary, queryable           | Text, line-based      |
| Query capability | Rich (time, level, service) | Limited (grep/regex)  |
| Persistence      | Automatic rotation          | Manual management     |
| Structure        | Metadata preserved          | Context lost          |

**Operational Advantage**: journalctl preserves more context and is harder to lose.

## Current Focus

Development of operational understanding of:

- Permission and ownership debugging workflows
- Service failure diagnosis patterns
- Process lifecycle management for infrastructure automation
- Integration between users, permissions, and service behavior

## Open Questions

- How do capabilities (Linux capabilities) differ from traditional permissions?
- What is the practical difference between socket activation vs traditional service startup?
- How does SELinux/AppArmor affect permission enforcement?
- What are best practices for systemd service hardening?

## References

- Linux `man` pages: `man ls`, `man chmod`, `man systemctl`, `man journalctl`
- systemd documentation: https://www.freedesktop.org/wiki/Software/systemd/
- Linux permission model: Core Unix permission semantics
