# GitHub Actions - Reusable Actions

This directory contains reusable composite actions for xword-dl workflows.

## Available Actions

### setup-ssh-proxy
**Purpose:** Establish an SSH SOCKS proxy tunnel for routing traffic through a remote server

**Inputs:**
- `ssh-proxy-host` (optional): SSH server hostname or IP. If not provided, the action does nothing.
- `ssh-proxy-port` (optional, default: "22"): SSH port
- `ssh-proxy-user` (optional, default: "ubuntu"): SSH username
- `ssh-private-key` (optional): SSH private key for authentication
- `socks-port` (optional, default: "1080"): Local SOCKS proxy port

**Side Effects:**
- Sets `HTTPS_PROXY` environment variable for all subsequent steps in the job
- Sets `SSH_TUNNEL_PID` environment variable with the tunnel process ID
- Creates `~/.ssh/xword_proxy_key` file (cleaned up by cleanup-ssh-proxy)

**Example Usage:**
```yaml
- name: Setup SSH Proxy
  uses: ./.github/actions/setup-ssh-proxy
  with:
    ssh-proxy-host: ${{ secrets.SSH_PROXY_HOST }}
    ssh-proxy-port: ${{ secrets.SSH_PROXY_PORT }}
    ssh-proxy-user: ${{ secrets.SSH_PROXY_USER }}
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
    socks-port: '1080'
```

---

### cleanup-ssh-proxy
**Purpose:** Tear down the SSH SOCKS proxy tunnel established by setup-ssh-proxy

**Inputs:** None

**Example Usage:**
```yaml
- name: Cleanup SSH Proxy
  if: always()
  uses: ./.github/actions/cleanup-ssh-proxy
```

**Note:** Use `if: always()` to ensure cleanup runs even if previous steps fail.

---

### run-xword-dl
**Purpose:** Run xword-dl with optional per-invocation SSH proxy setup

**Inputs:**
- `command` (required): The xword-dl command and arguments to run
- `ssh-proxy-host` (optional): SSH server hostname or IP
- `ssh-proxy-port` (optional, default: "22"): SSH port
- `ssh-proxy-user` (optional, default: "ubuntu"): SSH username
- `ssh-private-key` (optional): SSH private key for authentication
- `socks-port` (optional, default: "1080"): Local SOCKS proxy port
- `nyt-s-value` (optional): NYT-S cookie value for NYT authentication

**Features:**
- Sets up SSH tunnel if `ssh-proxy-host` is provided
- Automatically configures HTTPS_PROXY
- Handles NYT authentication if `nyt-s-value` is provided
- Cleans up tunnel after command completes

**Example Usage:**
```yaml
- name: Test outlet with proxy
  uses: ./.github/actions/run-xword-dl
  with:
    command: 'xword-dl tny'
    ssh-proxy-host: ${{ secrets.SSH_PROXY_HOST }}
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

**Note:** This action creates and destroys a tunnel for each invocation. For workflows that run many commands, consider using `setup-ssh-proxy` once at the job level instead.

---

## Usage Patterns

### Job-Level SSH Proxy (Recommended)

Use when running multiple commands that need the proxy:

```yaml
steps:
  - uses: actions/checkout@v4
  - name: Setup SSH Proxy
    uses: ./.github/actions/setup-ssh-proxy
    with:
      ssh-proxy-host: ${{ secrets.SSH_PROXY_HOST }}
      ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

  # All subsequent steps automatically use HTTPS_PROXY
  - name: Test 1
    run: xword-dl outlet1
  - name: Test 2
    run: xword-dl outlet2

  - name: Cleanup SSH Proxy
    if: always()
    uses: ./.github/actions/cleanup-ssh-proxy
```

### Per-Invocation SSH Proxy

Use when only specific commands need the proxy:

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Test without proxy
    run: xword-dl outlet1

  - name: Test with proxy
    uses: ./.github/actions/run-xword-dl
    with:
      command: 'xword-dl outlet2'
      ssh-proxy-host: ${{ secrets.SSH_PROXY_HOST }}
      ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Conditional Proxy Setup

The setup-ssh-proxy action automatically skips setup if `ssh-proxy-host` is not provided:

```yaml
- name: Setup SSH Proxy
  uses: ./.github/actions/setup-ssh-proxy
  with:
    # If SSH_PROXY_HOST secret is empty, this does nothing
    ssh-proxy-host: ${{ secrets.SSH_PROXY_HOST }}
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

# This step works with or without the proxy
- name: Test outlet
  run: xword-dl tny
```

## Security Considerations

- Always store SSH keys and credentials in GitHub Secrets
- Use `if: always()` on cleanup steps to prevent key files from persisting
- The SSH tunnel uses SOCKS5 protocol (`socks5h://`) which performs DNS resolution on the proxy server
- SSH keys are stored temporarily in `~/.ssh/xword_proxy_key` and removed during cleanup
