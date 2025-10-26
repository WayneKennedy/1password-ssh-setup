# 1Password Setup on Raspberry Pi (ARM64)

Complete guide for setting up 1Password with SSH agent on Raspberry Pi, eliminating local SSH key storage.

## Prerequisites

- Raspberry Pi running 64-bit OS (Debian/Raspberry Pi OS)
- Active 1Password account
- ARM64 architecture (verify with `uname -m` - should show `aarch64`)

## Installation Steps

### 1. Install 1Password CLI

```bash
# Add 1Password GPG key
curl -sS https://downloads.1password.com/linux/keys/1password.asc | \
  sudo gpg --dearmor --output /usr/share/keyrings/1password-archive-keyring.gpg

# Add 1Password repository
echo "deb [arch=arm64 signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/arm64 stable main" | \
  sudo tee /etc/apt/sources.list.d/1password.list

# Add signing policy
sudo mkdir -p /etc/debsig/policies/AC2D62742012EA22/
curl -sS https://downloads.1password.com/linux/debian/debsig/1password.pol | \
  sudo tee /etc/debsig/policies/AC2D62742012EA22/1password.pol

# Add debsig keyring
sudo mkdir -p /usr/share/debsig/keyrings/AC2D62742012EA22
curl -sS https://downloads.1password.com/linux/keys/1password.asc | \
  sudo gpg --dearmor --output /usr/share/debsig/keyrings/AC2D62742012EA22/debsig.gpg

# Install CLI
sudo apt update
sudo apt install -y 1password-cli
```

### 2. Configure 1Password CLI

Add to your `~/.bashrc`:

```bash
# 1Password CLI configuration
export OP_SECRET_KEY="your-secret-key-here"

# 1Password SSH Agent
export SSH_AUTH_SOCK=~/.1password/agent.sock

# Helper functions
op-get() {
    op item get "$1" --fields password
}

op-read() {
    op read "$1"
}
```

### 3. Sign in to 1Password CLI

```bash
# Add your account
op account add --address my.1password.eu --email your-email@example.com

# Sign in (creates session token)
eval $(op signin)
```

### 4. Install 1Password Desktop App

```bash
# Download ARM64 package
curl -sSO https://downloads.1password.com/linux/tar/stable/aarch64/1password-latest.tar.gz

# Extract
tar -xf 1password-latest.tar.gz

# Install
sudo mkdir -p /opt/1Password
sudo mv 1password-*/* /opt/1Password/
sudo /opt/1Password/after-install.sh
```

### 5. Enable SSH Agent in Desktop App

1. Launch 1Password: `1password`
2. Sign in to your account
3. Go to **Settings** → **Developer**
4. Enable **"Use the SSH agent"**
5. Enable **"Integrate with 1Password CLI"**

The app will automatically create `~/.ssh/config`:

```
Host *
	IdentityAgent ~/.1password/agent.sock
```

### 6. Set Up Auto-start

Create `~/.config/autostart/1password.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=1Password
Exec=/opt/1Password/1password --silent
Icon=1password
Comment=1Password Password Manager with SSH Agent
X-GNOME-Autostart-enabled=true
StartupNotify=false
Terminal=false
```

## Verification

Test SSH authentication without local keys:

```bash
# Test with GitHub
ssh -T git@github.com

# List available keys (requires app running)
ssh-add -l

# Verify no local keys exist
ls ~/.ssh/id_*  # Should show no results
```

## Key Points

- **No local SSH keys stored** - All keys managed in 1Password
- **Desktop app must be running** - SSH agent is part of the desktop app
- **Auto-starts on login** - Runs silently in system tray
- **Same workflow as Windows/Mac** - Consistent cross-platform experience
- **Biometric unlock** - If your Pi supports it (Touch ID, fingerprint, etc.)

## Troubleshooting

### SSH agent not working

```bash
# Check if 1Password is running
pgrep 1password

# Check if socket exists
ls -la ~/.1password/agent.sock

# Verify SSH_AUTH_SOCK is set
echo $SSH_AUTH_SOCK
```

### Connection refused

- Ensure 1Password desktop app is running
- The app can't be fully closed - must stay in system tray
- Restart the app if needed: `1password &`

### CLI session expired

```bash
# Re-authenticate (sessions expire after 30 min of inactivity)
eval $(op signin)
```

## Useful Commands

```bash
# List all items
op item list

# Get password from CLI
op item get "GitHub" --fields password

# List vaults
op vault list

# List SSH keys
op item list --categories "SSH Key"

# Read secret reference
op read "op://Private/GitHub/password"
```

## Security Benefits

1. **Single source of truth** - One SSH key across all devices
2. **No key sprawl** - Eliminate per-device SSH keys
3. **Centralized management** - Add/revoke keys from 1Password
4. **Audit trail** - 1Password logs when keys are used
5. **Secure storage** - Keys encrypted at rest and in transit
6. **Easy rotation** - Update key once, applies everywhere

## References

- [1Password for Linux](https://support.1password.com/install-linux/)
- [1Password SSH Agent](https://developer.1password.com/docs/ssh/agent/)
- [1Password CLI](https://developer.1password.com/docs/cli/)

---

**Note:** This guide assumes Raspberry Pi 4/5 or similar with ARM64 architecture. For other ARM devices, adjust the installation URLs accordingly.
