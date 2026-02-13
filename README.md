# MiaoWallet — SUI Wallet for OpenClaw

A custom OpenClaw skill for managing SUI wallets and on-chain transactions, with secure Keychain storage and Dry Run transaction preview.

---

## Features

- **Dry Run Preview** — Simulate transactions before signing, showing balance changes and gas estimates
- **SuiNS Domain Resolution** — Send to `name.sui` instead of raw addresses
- **macOS Keychain Integration** — Private keys stored securely, never in files
- **Interactive Wallet Panel** — Terminal-based management UI
- **Multi-Bot Support** — Shared via symlinks across OpenClaw instances
- **Custom Commands** — `-miaowallet` opens the wallet panel

---

## File Structure

```
skills/miao-wallet/
├── SKILL.md                  # Bot-facing documentation (how bots use this skill)
├── README.md                 # This file (human-facing documentation)
├── sui_transfer.py           # Transfer script with --dry-run and --yes modes
├── wallet_panel.py           # CLI wallet management (list/add/remove/test)
├── miaowallet.command        # macOS launcher — double-click to open wallet panel
├── add_wallet.command        # Wallet creation helper
├── wallet_mcp_server.py      # MCP server for tool integration
├── .wallet_addresses.json    # Cached addresses (no private keys, safe to share)
└── venv/                     # Python virtual environment
```

---

## Setup

### Prerequisites

- macOS with Keychain Access
- Python 3.10+
- OpenClaw installed

### Install

```bash
cd ~/.openclaw/skills/miao-wallet
python3 -m venv venv
source venv/bin/activate
pip install keyring pynacl bech32 requests
```

### Add a Wallet

Double-click `add_wallet.command` or run:

```bash
source venv/bin/activate
python3 wallet_panel.py add
```

This stores the private key in macOS Keychain under service `openclaw_bot`.

### Multi-Bot Sync

If running multiple OpenClaw instances, symlink the skill directory:

```bash
ln -s ~/.openclaw/skills ~/.openclaw-deepseek/skills
ln -s ~/.openclaw/skills ~/.openclaw-gemini/skills
```

All instances share the same tools. See [Multi-OpenClaw-Setup-on-Mac](../../../shared/github/Multi-OpenClaw-Setup-on-Mac.md) for details.

---

## Usage

### Transfer (Two-Step Flow)

**Step 1 — Preview (no Keychain needed):**

```bash
python3 sui_transfer.py sui1 recipient.sui 0.5 --dry-run
```

Output:
```
🔍 Resolving SuiNS: recipient.sui → 0xabc...def

📤 Transfer Preview (Dry Run)
   Wallet: sui1
   Sender: 0x123...456
   Amount: 0.5 SUI
   Balance: 2.5 SUI

=======================================================
  📋 Transaction Preview (Dry Run Simulation)
=======================================================
  ✅ Simulation: Success

  💰 Balance Changes:
     You (sender): -0.501548 SUI
     Recipient: +0.5 SUI

  ⛽ Gas Estimate:
     Computation: 0.00055 SUI
     Storage: 0.001976 SUI
     Rebate: -0.000978 SUI
     Total: ~0.001548 SUI

  📨 Summary:
     Transfer: 0.5 SUI
     Gas: ~0.001548 SUI
     Total cost: ~0.501548 SUI
=======================================================
```

**Step 2 — Execute (after user confirms):**

```bash
python3 sui_transfer.py sui1 recipient.sui 0.5 --yes
```

### How Dry Run Works

- Uses Sui RPC `sui_dryRunTransactionBlock` to simulate the transaction on-chain without signing
- Wallet addresses cached in `.wallet_addresses.json` so preview never touches Keychain
- Bot shows preview in chat → user says "confirm" → bot executes with `--yes`
- Matches the UX of wallet apps (Sui Wallet, Suiet) where you review before signing

### Wallet Panel

Open the interactive management panel:

```bash
open miaowallet.command    # Opens a Terminal window
```

Or via CLI:

```bash
source venv/bin/activate
python3 wallet_panel.py list          # List wallets
python3 wallet_panel.py add           # Add wallet
python3 wallet_panel.py remove <name> # Remove wallet
python3 wallet_panel.py test <name>   # Test wallet access
python3 wallet_panel.py reset-acl <name>  # Reset Keychain ACL
```

### Bot Commands

| Command | Action |
|---------|--------|
| `-miaowallet` | Open wallet panel window |
| `转 X SUI 给 <address/name.sui>` | Transfer flow (preview → confirm → execute) |

---

## Keychain Notes

Private keys are stored in macOS Keychain:

```
Service: openclaw_bot
Account: <wallet_alias> (e.g., sui1)
```

### Headless/Background Process Access

OpenClaw runs as a background process which may not have GUI permissions for Keychain popups. Solutions:

1. **Pre-authorize** — Open Keychain Access → find `openclaw_bot` → Access Control → add Python to "Always Allow"
2. **Terminal auth** — Run once in Terminal:
   ```bash
   security find-generic-password -s "openclaw_bot" -a "sui1" -w
   ```
   Click "Always Allow" on the popup. Future access won't prompt.
3. **Address caching** — `.wallet_addresses.json` stores addresses (not keys) so `--dry-run` never needs Keychain

---

## Security

- Private keys **never leave Keychain** — fetched only for signing, cleared from memory immediately after
- `.wallet_addresses.json` contains **only public addresses** (safe)
- `--dry-run` mode requires **zero secrets** — purely on-chain simulation
- Keys are **not stored in files, environment variables, or config**

---

## API Reference

### sui_transfer.py

```
Usage: python3 sui_transfer.py <wallet> <recipient> <amount> [options]

Arguments:
  wallet      Wallet alias (e.g., sui1)
  recipient   SUI address (0x...) or SuiNS domain (name.sui)
  amount      Amount in SUI (e.g., 0.5)

Options:
  --dry-run   Simulate only, don't sign or execute
  --yes       Skip confirmation, execute immediately
  (none)      Interactive mode — shows preview, asks for y/N confirmation
```
