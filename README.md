# Restricted User Management System

System for creating and managing restricted users with minimal permissions using rbash (restricted bash).

## Overview

Restricted users are created with **restricted bash (rbash)** — a restricted shell that:

- ❌ Cannot change PATH
- ❌ Cannot change directory (cd)
- ❌ Cannot redirect output (>, >>)
- ❌ Cannot use commands with `/` in path
- ❌ Has no access to system commands
- ✅ Has access **only** to explicitly allowed wrapper commands

## Installation

### Requirements

- Linux (any distribution)
- Root access (sudo)
- Bash shell

### Installation Steps

1. Download and run the installation script:

```bash
# Download install.sh to your server
wget https://example.com/install.sh  # Or use scp to copy it

# Run installation
sudo bash install.sh
```

2. Scripts will be installed to `/srv/scripts/`

3. Navigate to scripts directory:

```bash
cd /srv/scripts
```

## Usage

### Create Restricted User

```bash
cd /srv/scripts
sudo ./setup-user.sh
```

Enter the username when prompted. The script will:
- Create user with restricted bash shell
- Set up restricted environment
- Generate and set password automatically (for SSH access)
- Configure sudo for passwordless switching

### Add Allowed Command

```bash
sudo ./add-command.sh <command_name> "<full_command>"
```

**Important:** For commands **without sudo prefix**, you must use full paths to executables (e.g., `/usr/bin/df`, `/usr/bin/free`). For commands with `sudo` prefix, full paths are not required as sudo handles path resolution.

**Examples:**

```bash
# Switch to another user
sudo ./add-command.sh switch-appuser "sudo -iu appuser"

# Fix nginx permissions (with sudo - no full path needed)
sudo ./add-command.sh fix-nginx-perms "sudo chown -R nginx:nginx /srv"

# Restart service (with sudo - no full path needed)
sudo ./add-command.sh restart-nginx "sudo systemctl restart nginx"

# View logs (with sudo - no full path needed)
sudo ./add-command.sh view-logs "sudo tail -f /var/log/nginx/error.log"

# Deploy script (with sudo - no full path needed)
sudo ./add-command.sh deploy "sudo /opt/scripts/deploy.sh"

# Check disk (without sudo - full path required)
sudo ./add-command.sh check-disk "/usr/bin/df -h"

# Check memory (without sudo - full path required)
sudo ./add-command.sh check-memory "/usr/bin/free -h"
```

### List User Commands

```bash
sudo ./list-commands.sh
```

Enter username when prompted to see all allowed commands for that user.

### Remove Command

```bash
sudo ./remove-command.sh <command_name>
```

Enter username when prompted.

### List All Restricted Users

```bash
sudo ./list-users.sh
```

Shows all restricted users with their UID and number of allowed commands.

### Add SSH Public Key

```bash
sudo ./add-ssh-key.sh
```

Enter username and SSH public key when prompted. Useful for CI/CD systems (like GitLab Runner) that need SSH access.

**Example:**

```bash
sudo ./add-ssh-key.sh
# Enter username: runner
# Enter SSH public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ... user@host
```

### Remove Restricted User

```bash
sudo ./remove-user.sh
```

Enter the username when prompted. The script will:
- Remove all sudoers files
- Remove log directory
- Stop all user processes
- Remove user account and home directory

### Switch to Restricted User

```bash
sudo -iu <username>
```

## Complete Workflow Example

```bash
# Navigate to scripts directory
cd /srv/scripts

# Create user
sudo ./setup-user.sh  # Enter: runner

# Add SSH key (for CI/CD)
sudo ./add-ssh-key.sh  # Enter: runner, then paste SSH key

# Add commands
sudo ./add-command.sh restart-nginx "sudo systemctl restart nginx"
sudo ./add-command.sh restart-app "sudo systemctl restart myapp"
sudo ./add-command.sh deploy "sudo /opt/deploy/run.sh"

# List users
sudo ./list-users.sh

# List commands
sudo ./list-commands.sh  # Enter: runner

# Remove command
sudo ./remove-command.sh restart-nginx  # Enter: runner

# Remove user
sudo ./remove-user.sh  # Enter: runner
```

## Security

### What Restricted User CANNOT Do

| Action | Result |
|--------|--------|
| `cd /tmp` | `-rbash: cd: restricted` |
| `ls` | `-rbash: ls: command not found` |
| `cat /etc/passwd` | `-rbash: cat: command not found` |
| `/bin/ls` | `-rbash: /bin/ls: restricted: cannot specify commands with /` |
| `echo test > file` | `-rbash: file: restricted: cannot redirect output` |
| `PATH=/usr/bin:$PATH` | `-rbash: PATH: readonly variable` |

### Security Recommendations

1. **Use full paths for non-sudo commands**: Commands without `sudo` prefix must use full paths (e.g., `/usr/bin/df` instead of `df`)
2. **Minimal permissions**: Add only necessary commands
3. **Specific paths**: Use specific paths in commands (`/srv` instead of `*`)
4. **Audit**: Regularly check logs via `sudo journalctl -t <username>-<command_name>`
5. **Review**: Periodically review list of allowed commands

## Logging

All command executions are logged to **syslog** with tag `<username>-<command_name>`:

```bash
# View logs
sudo journalctl -t <username>-<command_name>

# Or via syslog
sudo grep "<username>-" /var/log/syslog | tail -20
```

## Troubleshooting

### "command not found" when logging in

Command is not added. Check:
```bash
sudo ./list-commands.sh
```

### Command execution prompts for password

Check sudoers file:
```bash
sudo visudo -c -f /etc/sudoers.d/<username>-<command_name>
```

### Verify rbash restrictions

```bash
sudo -iu <username>
# Try restricted commands (all should fail):
cd /tmp          # restricted
ls               # command not found
/bin/bash        # restricted: cannot specify '/'
```

### SSH key not working

Check permissions:
```bash
sudo ls -la /home/<username>/.ssh/
```

Should be:
- `.ssh` directory: `drwx------` (700)
- `authorized_keys`: `-rw-------` (600)
- Both owned by `<username>:<username>`

## Project Files

Scripts are located in `/srv/scripts/`:

| File | Description |
|------|-------------|
| `setup-user.sh` | Create restricted user |
| `add-command.sh` | Add allowed command |
| `remove-command.sh` | Remove allowed command |
| `list-commands.sh` | List all commands for user |
| `list-users.sh` | List all restricted users |
| `add-ssh-key.sh` | Add SSH public key to user |
| `remove-user.sh` | Remove restricted user |
