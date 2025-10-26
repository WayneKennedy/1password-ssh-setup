# 1Password Setup on Windows + WSL

Complete guide for setting up 1Password with SSH agent on Windows and Windows Subsystem for Linux (WSL), eliminating local SSH key storage.

## Prerequisites

- Windows 10/11 (64-bit)
- Active 1Password account
- Windows Subsystem for Linux (WSL2 recommended) - Optional but recommended for development work

## Installation Steps

### 1. Install 1Password Desktop App for Windows

1. Download from: https://1password.com/downloads/windows/
2. Run the installer
3. Sign in to your 1Password account
4. Let it import passwords from your browser (optional)

### 2. Enable SSH Agent in 1Password

1. Open **1Password Desktop**
2. Go to **Settings** → **Developer**
3. Enable **"Use the SSH agent"**
4. Enable **"Display key names when authorizing connections"**
5. Choose **"Use Key Names"** when prompted (stores unencrypted key names locally for easier identification)
6. Enable **"Integrate with 1Password CLI"** (if you want CLI access)

### 3. Disable Windows OpenSSH Agent (Avoid Conflicts)

The Windows built-in SSH agent conflicts with 1Password's agent. Disable it:

**Option A: PowerShell (Admin)**
```powershell
Stop-Service ssh-agent
Set-Service -Name ssh-agent -StartupType Disabled
```

**Option B: Services GUI**
1. Press `Win + R`
2. Type `services.msc` and press Enter
3. Find **"OpenSSH Authentication Agent"**
4. Right-click → **Properties**
5. Set **Startup type** to **"Disabled"**
6. Click **Stop** → **OK**

### 4. Import Your SSH Key into 1Password

#### Option A: Import Existing Key

1. In 1Password Desktop, click **"+"** (New Item)
2. Select **"SSH Key"** category
3. Give it a descriptive name (e.g., "WK ED25519 Identity")
4. Click **"Replace Private Key"**
5. Paste your private key (from `~/.ssh/id_ed25519` or similar)
6. Add notes about where this key is used (GitHub, servers, etc.)
7. Click **Save**

#### Option B: Generate New Key in 1Password

1. In 1Password Desktop, click **"+"** (New Item)
2. Select **"SSH Key"** category
3. Give it a descriptive name
4. Click **"Generate SSH Key"**
5. Choose **ED25519** (recommended for security and speed)
6. Add notes about intended use
7. Click **Save**
8. Click **"Copy Public Key"** and upload to GitHub/servers

**Upload Public Key to Services:**
- **GitHub**: Settings → SSH and GPG keys → New SSH key
- **GitLab**: Preferences → SSH Keys → Add new key
- **Servers**: Append to `~/.ssh/authorized_keys`

### 5. Test SSH Authentication (Windows)

Open PowerShell or Windows Terminal:

```powershell
# Test GitHub
ssh -T git@github.com

# Test GitLab
ssh -T git@gitlab.com

# List available keys (from 1Password)
ssh-add -l
```

You should see a 1Password authorization prompt the first time.

## WSL Integration (For Developers)

If you use WSL for development, configure it to use Windows SSH with 1Password's agent:

### 1. Configure Git to Use Windows SSH

In WSL terminal:

```bash
# Tell Git to use Windows SSH client (which connects to 1Password)
git config --global core.sshCommand "ssh.exe"
```

### 2. Set Up Convenient Aliases

Add to `~/.bashrc`:

```bash
# 1Password SSH agent integration (Windows SSH + 1Password)
# Use Windows SSH binaries which connect to 1Password SSH agent
alias ssh='ssh.exe'
alias ssh-add='ssh-add.exe'
```

Reload your shell:

```bash
source ~/.bashrc
```

### 3. Test from WSL

```bash
# Test GitHub
ssh -T git@github.com

# Test GitLab
ssh -T git@gitlab.com

# Git operations
git clone git@github.com:username/repo.git
```

## Verification

Confirm everything works and no local keys exist:

### On Windows (PowerShell)

```powershell
# List keys from 1Password
ssh-add -l

# Verify SSH_AUTH_SOCK points to 1Password
$env:SSH_AUTH_SOCK

# Test authentication
ssh -T git@github.com
```

### On WSL (Bash)

```bash
# Test authentication
ssh -T git@github.com

# Verify no local private keys
ls ~/.ssh/id_* 2>/dev/null  # Should return nothing

# List what IS in .ssh (should only be known_hosts)
ls -la ~/.ssh/
```

## Remove Local SSH Keys (Security Best Practice)

Once 1Password is working, remove local SSH keys:

### Backup First (Optional but Recommended)

```bash
# WSL/Linux
mkdir -p ~/ssh-backup-$(date +%Y%m%d)
cp -r ~/.ssh/* ~/ssh-backup-$(date +%Y%m%d)/

# Windows PowerShell
mkdir $env:USERPROFILE\ssh-backup-$(Get-Date -Format "yyyyMMdd")
Copy-Item $env:USERPROFILE\.ssh\* $env:USERPROFILE\ssh-backup-$(Get-Date -Format "yyyyMMdd")\ -Recurse
```

### Remove Private Keys

```bash
# WSL/Linux
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
rm ~/.ssh/id_rsa ~/.ssh/id_rsa.pub  # If you had RSA keys

# Windows PowerShell
Remove-Item $env:USERPROFILE\.ssh\id_ed25519*
Remove-Item $env:USERPROFILE\.ssh\id_rsa*
```

**Note:** Keep `known_hosts` files - they only store server fingerprints, not secrets.

## Git Commit Signing (Optional)

Sign your commits with SSH keys from 1Password:

### Configure Git Signing

```bash
# Set your SSH key for signing (get public key from 1Password)
git config --global user.signingkey "ssh-ed25519 AAAAC3Nza..."

# Enable commit signing
git config --global commit.gpgsign true
git config --global gpg.format ssh

# For WSL: Use Windows SSH
git config --global gpg.ssh.program "ssh.exe"
```

### Add Signing Key to GitHub

1. Copy your SSH public key from 1Password
2. GitHub → Settings → SSH and GPG keys → New SSH key
3. Select **"Signing Key"** as key type
4. Paste and save

## Key Points

- **No local SSH keys stored** - All keys managed in 1Password
- **1Password Desktop must be running** - SSH agent is part of the desktop app
- **WSL uses Windows SSH** - No complex bridging needed
- **Biometric unlock** - Windows Hello, fingerprint, or master password
- **Cross-platform sync** - Same key available on all your devices (once 1Password installed)
- **Session-based authorization** - Each new terminal session requires approval

## Troubleshooting

### SSH agent not working

**Check if 1Password is running:**
```powershell
Get-Process 1Password
```

**Verify SSH agent is enabled:**
- 1Password → Settings → Developer → "Use the SSH agent" (should be checked)

### "Permission denied (publickey)" error

1. Check 1Password has your SSH key stored
2. Ensure you approved the 1Password authorization prompt
3. Verify the public key is uploaded to the service (GitHub, etc.)
4. Test connection: `ssh -T git@github.com`

### WSL can't find ssh.exe

Ensure Windows OpenSSH is installed:

```powershell
# In PowerShell (Admin)
Add-WindowsCapability -Online -Name OpenSSH.Client
```

### Git operations hang or prompt for password

Your Git remote might be using HTTPS instead of SSH:

```bash
# Check remote URL
git remote -v

# If it shows https://, switch to SSH
git remote set-url origin git@github.com:username/repo.git
```

### 1Password authorization prompt doesn't appear

1. Ensure 1Password Desktop is running (not just browser extension)
2. Check 1Password isn't minimized to system tray
3. Try restarting 1Password Desktop
4. Verify you're signed in to 1Password

## Useful Commands

```bash
# List SSH keys from 1Password
ssh-add -l

# List keys with full fingerprints
ssh-add -L

# Test SSH connection with verbose output
ssh -Tv git@github.com

# Check Git SSH configuration
git config --get core.sshCommand
git config --get gpg.ssh.program
```

## Security Benefits

1. **Single source of truth** - One SSH key across all devices
2. **No key sprawl** - Eliminate per-device SSH keys
3. **Centralized management** - Add/revoke keys from 1Password
4. **Audit trail** - 1Password logs when keys are used
5. **Secure storage** - Keys encrypted at rest and in transit
6. **Easy rotation** - Update key once, applies everywhere
7. **Biometric unlock** - Faster than typing passwords
8. **Cross-platform** - Same key on Windows, Linux, Mac, mobile

## Migration from Existing SSH Keys

If you're migrating from existing SSH keys:

1. **Audit what you have:**
   ```bash
   ls -la ~/.ssh/
   ```

2. **Identify your keys:**
   - Personal keys (e.g., `id_ed25519`, `id_rsa`)
   - Service keys (e.g., `github_actions`, `deploy_key`)

3. **Import personal keys to 1Password** (see Step 4 above)

4. **Service keys:**
   - These can also go in 1Password
   - Or regenerate them fresh and store only in service secrets
   - Example: GitHub Actions doesn't need keys on your local machine

5. **Test everything works** before deleting local copies

6. **Remove local keys** only after successful testing

## Advanced: Using npiperelay (Alternative Method)

The method above (using Windows `ssh.exe` from WSL) is simplest and recommended. However, if you need native Linux SSH client behavior, you can bridge 1Password's Windows agent to WSL using `npiperelay` + `socat`. This is more complex and usually unnecessary.

See: https://developer.1password.com/docs/ssh/integrations/wsl/ for details.

## References

- [1Password for Windows](https://support.1password.com/install-windows/)
- [1Password SSH Agent](https://developer.1password.com/docs/ssh/agent/)
- [1Password WSL Integration](https://developer.1password.com/docs/ssh/integrations/wsl/)
- [1Password CLI](https://developer.1password.com/docs/cli/)
- [GitHub SSH Keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)

---

**Note:** This guide was tested on Windows 11 with WSL2 (Ubuntu 22.04) and 1Password 8.11.14.
