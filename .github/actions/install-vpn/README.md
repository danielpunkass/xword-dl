# Install VPN Action

A reusable composite action that installs OpenVPN and creates `open-vpn` and `close-vpn` commands that can be called from within workflow scripts.

## Usage

```yaml
- name: Install VPN
  uses: ./.github/actions/install-vpn
  with:
    openvpn_config: ${{ secrets.OPENVPN_CONFIG }}

- name: Use VPN commands in your scripts
  run: |
    # Connect to VPN
    open-vpn

    # Do something over VPN
    curl https://geo-restricted-site.com

    # Disconnect from VPN
    close-vpn
```

## Inputs

| Input | Description | Required |
|-------|-------------|----------|
| `openvpn_config` | Base64-encoded OpenVPN configuration file (.ovpn) | Yes |

## Outputs

| Output | Description |
|--------|-------------|
| `vpn_scripts_path` | Path to directory containing VPN helper scripts (`/tmp/vpn`) |

## Prerequisites

### GitHub Secrets

You need to set up the following GitHub secret:

- **OPENVPN_CONFIG**: Base64-encoded OpenVPN configuration file

To generate this secret value from your `.ovpn` file:

```bash
base64 -i your-config.ovpn | tr -d '\n' && echo
```

Copy the output and add it as a GitHub secret named `OPENVPN_CONFIG` in your repository settings.

### Runner Requirements

- This action is designed to run on `ubuntu-latest` runners
- Requires `sudo` privileges (available on GitHub-hosted runners)

## What This Action Does

1. **Installs OpenVPN** if not already present
2. **Decodes and saves** the base64-encoded configuration to `/tmp/vpn/client.ovpn`
3. **Creates helper scripts** that are added to PATH:
   - `open-vpn` - Connects to the VPN
   - `close-vpn` - Disconnects from the VPN
4. **Adds `/tmp/vpn` to PATH** so commands are available in all subsequent steps

## Available Commands

### `open-vpn`

Connects to the VPN using the configured OpenVPN profile.

**Features:**
- Idempotent (safe to call multiple times)
- Waits up to 30 seconds for connection
- Shows external IP after connecting
- Returns exit code 0 on success, 1 on failure

**Example:**
```bash
open-vpn
```

### `close-vpn`

Disconnects from the VPN.

**Features:**
- Idempotent (safe to call if not connected)
- Waits up to 10 seconds for disconnection
- Returns exit code 0 on success, 1 on failure

**Example:**
```bash
close-vpn
```

## Example Workflows

### Example 1: Simple VPN Usage

```yaml
name: Download with VPN

on: [workflow_dispatch]

jobs:
  download:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install VPN
        uses: ./.github/actions/install-vpn
        with:
          openvpn_config: ${{ secrets.OPENVPN_CONFIG }}

      - name: Download content
        run: |
          open-vpn
          curl https://example.com/content
          close-vpn
```

### Example 2: Selective VPN Usage Within a Script

```yaml
name: Mixed VPN Usage

on: [workflow_dispatch]

jobs:
  download:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install VPN
        uses: ./.github/actions/install-vpn
        with:
          openvpn_config: ${{ secrets.OPENVPN_CONFIG }}

      - name: Complex workflow
        run: |
          echo "Step 1: Without VPN"
          curl https://api.ipify.org

          echo "Step 2: Connect and download"
          open-vpn
          curl https://geo-restricted-content.com/file1
          curl https://geo-restricted-content.com/file2

          echo "Step 3: Disconnect"
          close-vpn

          echo "Step 4: Continue without VPN"
          curl https://api.ipify.org

      - name: Cleanup
        if: always()
        run: close-vpn || true
```

### Example 3: Multiple Downloads with Error Handling

```yaml
- name: Download puzzles with VPN
  run: |
    set -e  # Exit on error

    open-vpn

    # Download multiple items
    for outlet in nyt wsj guardian; do
      echo "Downloading $outlet..."
      uv run xword-dl "$outlet" --latest || echo "Failed: $outlet"
    done

    close-vpn
```

## Notes

- Commands are idempotent - safe to call multiple times
- VPN connection persists across multiple run steps
- Uses certificate-based authentication (no username/password required)
- Helper scripts check connection status before acting
- Always call `close-vpn` in cleanup steps with `if: always()` to ensure disconnection

## Troubleshooting

If VPN connection fails:
1. Check the OpenVPN logs: `cat /tmp/vpn/openvpn.log`
2. Verify your secret is correctly base64-encoded
3. Ensure your OpenVPN config has embedded certificates

If VPN doesn't disconnect:
1. Manually kill OpenVPN: `sudo killall openvpn`
2. Clean up files: `rm -rf /tmp/vpn`
