# Obsidian Git Sync Automation

This repository contains a script to automatically synchronize an Obsidian Vault with a remote Git repository. The synchronization includes pulling the latest changes, committing local modifications, and pushing updates to the remote repository.

## Features
- Automatically pulls the latest changes from the repository.
- Commits and pushes changes only if modifications are detected.
- Runs every 5 minutes using `cron`.
- Executes once at system startup using `@reboot` in `cron`.
- Uses a GitHub Personal Access Token (PAT) for authentication (recommended to store securely).

## Prerequisites
### Install Git & Cron
Ensure Git and Cron are installed on your system:
```sh
sudo apt install git cron   # Debian-based systems
sudo dnf install git cronie # Fedora-based systems
```

Enable and start Cron:
```sh
sudo systemctl enable cronie --now
```

## Setup
### Clone the Repository
```sh
git clone https://github.com/your-username/ObsidianVault.git
cd ObsidianVault
```

### Store GitHub Token Securely
It is recommended **not to store the token directly in the script**. Instead, use one of the following methods:

#### 1. Environment Variable
Add the following line to `~/.bashrc` or `~/.zshrc`:
```sh
export GITHUB_TOKEN="your-token-here"
```
Reload the shell:
```sh
source ~/.bashrc  # or source ~/.zshrc
```
Modify the script to use:
```sh
GIT_CREDENTIALS="https://your-username:${GITHUB_TOKEN}@github.com/your-username/ObsidianVault.git"
```

#### 2. `.netrc` File
Create a `~/.netrc` file with:
```
machine github.com
login your-username
password your-token-here
```
Set secure permissions:
```sh
chmod 600 ~/.netrc
```

## Script Configuration
Create a script file `obsidian_git_sync.sh`:
```sh
#!/bin/bash

REPO_PATH="$HOME/Documents/Obsidian Vault"
GITHUB_USER="Name"
GITHUB_TOKEN="Token"
GIT_REMOTE="https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/${GITHUB_USER}/ObsidianVault.git" 
BRANCH="main"

# Change to the repository directory
cd "$REPO_PATH" || { echo "Directory not found!"; exit 1; }

# If the repository is not linked, set the remote URL
if ! git remote get-url origin &>/dev/null; then
    git remote set-url origin "$GIT_REMOTE"
fi

# Stash changes before pulling
git stash push -m "Backup before pull"

# Fetch the latest changes
git pull --rebase --autostash origin "$BRANCH"

# Restore stashed changes
git stash pop || echo "No stashed changes found."

# Check if there are unstaged or staged changes
if ! git diff --quiet || ! git diff --cached --quiet; then
    echo "Changes detected – committing..."
    
    # Stage all changes
    git add .
    
    # Commit with the current date & time
    git commit -m "Auto-commit $(date '+%Y-%m-%d %H:%M:%S')"
    
    # Push changes
    git push origin "$BRANCH"
else
    echo "No changes - push skipped."
fi

```
Make the script executable:
```sh
chmod +x obsidian_git_sync.sh
```

## Automating with Cron
### Edit Cron Jobs
Run:
```sh
crontab -e
```
Add the following lines at the end:
```
*/5 * * * * /path/to/obsidian_git_sync.sh >> /path/to/git_sync.log 2>&1
@reboot sleep 60 && /path/to/obsidian_git_sync.sh >> /path/to/git_sync.log 2>&1
```
### Explanation
- `*/5 * * * *` → Runs the script every 5 minutes.
- `@reboot sleep 60 && ...` → Executes once after system startup, waiting 60 seconds.

## Ensuring Cron Runs After Reboot
To verify Cron is active after reboot:
```sh
systemctl status cronie  # or systemctl status cron
```
If not enabled:
```sh
sudo systemctl enable --now cronie  # or sudo systemctl enable --now cron
```

## Running the Script on Shutdown
To execute the script when the system shuts down, create a systemd service:

1. Create a file `/etc/systemd/system/obsidian_git_shutdown.service`:
```ini
[Unit]
Description=Obsidian Git Sync on Shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
ExecStart=/bin/true
ExecStop=/path/to/obsidian_git_sync.sh
RemainAfterExit=true

[Install]
WantedBy=halt.target reboot.target shutdown.target
```
2. Enable the service:
```sh
sudo systemctl enable obsidian_git_shutdown.service
```

## Logs & Debugging
To check the log output:
```sh
tail -f /path/to/git_sync.log
```
To debug Cron:
```sh
journalctl -u cronie --since "1 hour ago"  # or journalctl -u cron
```

## Conclusion
This setup ensures seamless synchronization of your Obsidian Vault to a Git repository at regular intervals, at startup, and optionally at shutdown. By following security best practices, you can safely authenticate using a token without exposing sensitive credentials.

---
**Note:** Adjust file paths and repository names according to your setup.

