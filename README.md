![pass-bitcoind](banner.webp)

# pass-bitcoind

A `pass` CLI extension for Bitcoin Core wallet backup and restore. Provides proper wallet descriptor backup/restore functionality using JSON descriptor export for secure storage in your password store. **Security-hardened** for production use with Bitcoin wallets containing funds.

**Version: 0.2.2**

## Features

- **🔒 Security-Hardened**: Secure temporary file handling (including RAM-disk preference), proper cleanup, and restrictive permissions.
- **🌐 Remote Server Compatible**: Works with remote Bitcoin Core servers via RPC calls.
- **📊 Descriptor-Based**: Uses Bitcoin Core's modern descriptor wallet format.
- **🔐 Encrypted Storage**: JSON descriptor files encrypted with GPG via `pass`.
- **🔄 Full Integration**: Complete Bitcoin Core descriptor wallet management.
- **📁 Import/Export**: Handle descriptor JSON files with secure permissions.
- **✅ Safe Operations**: Wallet validation, backup verification, and confirmation prompts.
- **🛠️ Complete Workflow**: Create, backup, restore, import, export, destroy, and list operations.

## Why This Approach?

`pass-bitcoind` (v0.2.0+) uses Bitcoin Core's modern descriptor-based approach which works with remote servers:

- **RPC-based**: Works with remote Bitcoin Core servers (no local file access needed).
- **Descriptor wallets**: Uses Bitcoin Core's modern descriptor wallet format (default in recent versions).
- **Complete backup**: Exports all wallet descriptors (including private keys) as JSON.
- **Native compatibility**: Uses Bitcoin Core's built-in RPC commands (`listdescriptors`, `importdescriptors`).
- **Production ready**: Secure descriptor export/import via `bitcoin-cli`.

## Installation

1.  Ensure your pass extensions directory exists:
    ```bash
    mkdir -p ~/.password-store/.extensions
    ```
2.  Copy the extension to your pass extensions directory:
    ```bash
    cp bitcoind.bash ~/.password-store/.extensions/bitcoind.bash # Assuming the script is named bitcoind.bash
    chmod +x ~/.password-store/.extensions/bitcoind.bash
    ```
3.  Enable pass extensions (if not already enabled):
    Add these lines to your shell profile (e.g., `~/.zshrc`, `~/.bashrc`, or `~/.profile`):
    ```bash
    export PASSWORD_STORE_ENABLE_EXTENSIONS=true
    # The PASSWORD_STORE_EXTENSIONS_DIR defaults to ~/.password-store/.extensions,
    # so explicitly setting it is often not necessary if using the default.
    # export PASSWORD_STORE_EXTENSIONS_DIR=~/.password-store/.extensions
    ```
    Then source your profile or open a new terminal.

## Dependencies

**Required:**

- `pass` - The standard Unix password manager (must be initialized).
- `bitcoin-cli` - Bitcoin Core command-line interface (configured for your server).
- `jq` - Command-line JSON processor.

**Optional (for enhanced functionality):**

- `bc` - Basic calculator (for wallet balance warnings).
- `shred` - Securely overwrite files (for enhanced temporary file cleanup).

**Install on macOS:**

```bash
brew install pass bitcoin jq bc coreutils # coreutils for shred
```

**Install on Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install pass bitcoin-utils jq bc secure-delete # bitcoin-utils provides bitcoin-cli, secure-delete provides shred
```

_(Note: Package names for `bitcoin-cli` and `shred` might vary slightly across distributions or Bitcoin Core installation methods.)_

**Setup Bitcoin Core:**

Make sure Bitcoin Core (`bitcoind`) is running and accessible via `bitcoin-cli`:

```bash
# For local server (example)
bitcoind -daemon

# For remote server, ensure your ~/.bitcoin/bitcoin.conf (or equivalent)
# is configured correctly for RPC access, e.g.:
# rpcuser=your_rpc_user
# rpcpassword=your_rpc_password
# rpcconnect=remote_server_ip
# rpcallowip=your_client_ip (if needed)

# Verify bitcoin-cli connectivity
bitcoin-cli getblockchaininfo
```

## Usage

### Create a new wallet and backup

```bash
pass bitcoind create mywallet
```

This creates a new Bitcoin Core descriptor wallet and immediately backs up all its descriptors to `pass`.

### Backup existing Bitcoin Core wallet

```bash
pass bitcoind backup existingwallet
```

### Restore wallet from pass to Bitcoin Core

```bash
# Restore with the same name as in pass
pass bitcoind restore mywallet

# Restore with a new name in Bitcoin Core
pass bitcoind restore mywallet newwalletname
```

### Import existing descriptor JSON file to `pass`

```bash
pass bitcoind import mywalletfromfile ~/path/to/wallet_backup.json
```

This imports a pre-existing JSON backup (conforming to the script's expected format) into `pass`.

### Export wallet from `pass` to a JSON file

```bash
pass bitcoind export mysecretwallet ~/backups/mysecretwallet_export.json
```

The exported file will have `600` permissions.

### List wallets

```bash
pass bitcoind list
```

Shows wallets stored in `pass` (under the configured prefix) and wallets currently loaded in Bitcoin Core.

### Remove wallet (from `pass` and unload from Bitcoin Core)

```bash
pass bitcoind destroy oldwallet
```

**Note on `destroy`**:
The `destroy` command:

1.  Removes the wallet backup from `pass` storage.
2.  Unloads the wallet from Bitcoin Core using `bitcoin-cli unloadwallet`.
    It does **not** delete the actual wallet files from the Bitcoin Core server's filesystem (e.g., files in `.bitcoin/wallets/` directory or your custom wallet directory). To completely remove wallet files from the server, you must manually delete them from the Bitcoin Core data directory after running the `destroy` command and ensuring the wallet is no longer in use.

### Show help

```bash
pass bitcoind help
```

## Workflow

### Backup Workflow

1.  **Create/Backup Command**:
    - For `create`: `bitcoin-cli createwallet` (creates a new descriptor wallet).
    - `bitcoin-cli -rpcwallet=<wallet-name> listdescriptors true` exports all wallet descriptors, including private keys.
    - A JSON object containing these descriptors and metadata (version, timestamp, wallet info) is constructed.
2.  **Storage**: The JSON data is piped to `pass insert -m <prefix>/<wallet-name>`, encrypting it with GPG and storing it.
3.  **Verification**: The stored backup is immediately read back from `pass` and validated.

### Restore Workflow

1.  **Restore Command**: `pass show <prefix>/<wallet-name>` retrieves the encrypted JSON data.
2.  The JSON is parsed to extract the necessary descriptor information.
3.  `bitcoin-cli createwallet <new-wallet-name> ... true` creates a new, empty, descriptor-enabled, blank wallet.
4.  `bitcoin-cli -rpcwallet=<new-wallet-name> importdescriptors '[{...}]'` imports all descriptors with private keys into the new wallet.
5.  **Result**: A full descriptor wallet with all addresses and private keys is restored and active in Bitcoin Core. A rescan will occur in the background.

## Environment Variables

- **`PASS_BITCOIND_PREFIX`**: Changes the subdirectory within `pass` where wallet backups are stored.
  Default: `bitcoind`
- **`TMPDIR`**: Can be set to influence where `mktemp` creates temporary files if `/dev/shm` is not available/writable. The script prioritizes `/dev/shm`.

### Examples with custom prefix

```bash
# Store wallets in a custom folder within pass
PASS_BITCOIND_PREFIX=bitcoin_wallets pass bitcoind create mymainwallet

# Organize by purpose
PASS_BITCOIND_PREFIX=cold_storage pass bitcoind create offline_backup
PASS_BITCOIND_PREFIX=hot_wallets pass bitcoind create daily_spender
```

## Storage Format

Wallets are stored in `pass` as GPG-encrypted JSON files. The path structure within your password store will be:

```
<your-password-store-dir>/
└── <PASS_BITCOIND_PREFIX>/
    ├── mywallet.gpg
    ├── testnet-wallet.gpg
    └── cold-storage-backup.gpg
```

### JSON Format Example (content of the decrypted file)

```json
{
  "version": "0.2.2",
  "wallet_name": "mywallet",
  "timestamp": 1748448121,
  "wallet_info": {
    "walletname": "mywallet",
    "walletversion": 180000,
    "format": "sqlite",
    "descriptors": true,
    "avoid_reuse": false,
    "keypoolsize": 1000,
    "keypoolsize_hd_internal": 1000,
    "paytxfee": 0.0,
    "private_keys_enabled": true,
    "external_signer": false
  },
  "descriptors": {
    "wallet_name": "mywallet",
    "descriptors": [
      {
        "desc": "wpkh([abcdef12/84h/0h/0h]xpub.../0/*)#checksum",
        "timestamp": "now",
        "active": true,
        "internal": false,
        "range": [0, 999],
        "next": 0
      },
      {
        "desc": "wpkh([abcdef12/84h/0h/0h]xpub.../1/*)#checksum",
        "timestamp": "now",
        "active": true,
        "internal": true,
        "range": [0, 999],
        "next": 0
      }
      // ... other descriptors if any (e.g. for miniscript, taproot)
    ]
  }
}
```

_(Actual content, especially `wallet_info` and `descriptors`, will vary based on your Bitcoin Core version and wallet type.)_

## Security Considerations

- **Master GPG Key**: The security of these backups relies entirely on the security of your `pass` GPG key. Protect it well!
- **RPC Security**: Ensure your Bitcoin Core RPC interface is properly secured (strong `rpcauth` or `rpcpassword`, `rpcallowip`, firewall rules, consider Tor or VPN for remote access). Private keys are transmitted over this connection.
- **Temporary Files**: The script uses `mktemp` to create temporary files, preferably in `/dev/shm` (RAM disk). These files are `chmod 0600` and are securely cleaned up using `shred` (if available) then `rm`. A `trap` ensures cleanup on script exit/interruption.
- **Verification**: Backups are verified after being written to `pass`, and imported/exported files are checked for basic integrity.
- **Wallet Unloading**: The `destroy` command only unloads the wallet. For complete deletion from the server disk, manual intervention is required.
- **`set -euo pipefail`**: The script uses strict error checking.

## Advantages Over Manual `dumpwallet`/`backupwallet.dat`

- **Remote Server Native**: Designed for RPC, doesn't require direct filesystem access to the Bitcoin Core server, unlike `backupwallet.dat`.
- **Descriptor Focused**: Aligns with modern Bitcoin Core descriptor wallets, ensuring all necessary information for address derivation and key management is backed up.
  `dumpwallet` is for legacy non-HD wallets or individual keys and is not a complete backup for descriptor wallets.
- **Selective Restore**: Descriptors can be imported into a new wallet without affecting other wallets on the server.
- **Human-Readable (ish) Backup**: The JSON format (when decrypted) is more inspectable than a binary `.dat` file.
- **Platform Independent Backup**: JSON descriptors are portable across systems.

## License

MIT License
