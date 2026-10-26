# LA TANDA

<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/46ef6f9d-ffc4-46e5-8e32-5a91b9401c66" />

# La Tanda Testnet Validator Guide (with Cosmovisor)

### RPC: https://latanda-rpc.zeycanode.com/

A practical, field-tested guide for running a validator on the **La Tanda Chain** testnet (`latanda-testnet-1`, Cosmos SDK v0.53.6 / CometBFT) under **Cosmovisor**, including the Discord verification steps (`!verify` and the HTTP-nonce infrastructure check) and the mistakes we hit along the way.

> **Disclaimer.** This is a community guide, not an official one. It builds on the public UTSA community guide and the La Tanda team's forum posts, and adds fixes for problems found in real use. Testnet LTD has no monetary value according to the project. Programs, endpoints and policies can change, so always check the pinned messages in the project's Discord. Last tested: October 2026.

## Contents

1. [Quick facts](#1-quick-facts)
2. [Requirements](#2-requirements)
3. [Variables](#3-variables)
4. [Pre-flight checks](#4-pre-flight-checks)
5. [Build the binary](#5-build-the-binary)
6. [Cosmovisor layout](#6-cosmovisor-layout)
7. [Init and genesis](#7-init-and-genesis)
8. [Configuration](#8-configuration)
9. [State sync](#9-state-sync)
10. [systemd service](#10-systemd-service)
11. [Wallet and faucet](#11-wallet-and-faucet)
12. [Create the validator](#12-create-the-validator)
13. [Discord verification](#13-discord-verification)
14. [Monitoring](#14-monitoring)
15. [Upgrades with Cosmovisor](#15-upgrades-with-cosmovisor)
16. [Backups and security](#16-backups-and-security)
17. [Troubleshooting](#17-troubleshooting)
18. [Credits](#18-credits)

## 1. Quick facts

| Item | Value |
| --- | --- |
| Chain ID | `latanda-testnet-1` |
| Binary / CLI | `latandad` |
| Home directory | `~/.latanda` |
| Denom | `ultd` (**1 LTD = 1,000,000 ultd**, 6 decimals) |
| Address prefix | `ltd` (validator operator: `ltdvaloper`, consensus: `ltdvalcons`) |
| Block time | about 5 seconds |
| Minimum gas price | `0.001ultd` (a typical fee is `5000ultd`) |
| Faucet | Discord, 10 LTD per person, once (see step 11) |
| Validator stake | no protocol minimum, active set up to 100 validators |
| Genesis | `https://latanda.online/chain/genesis.json` |
| Genesis SHA-256 | `98fc9871d6a3b7b12b3f7fcaa1ca3303ffcfad0f209d61355975a15069ac3907` |

**Unit warning.** The upstream guide's example uses an amount of `1100000000000000000ultd`. That assumes 18 decimals, but the chain uses 6. With the 10 LTD faucet drip (10,000,000 ultd) that transaction fails with `insufficient funds`. Use `1000000ultd` (1 LTD) instead. More on this in step 12.

## 2. Requirements

- Ubuntu 22.04 or newer, 2 vCPU, 4 GB RAM, 100 GB SSD is plenty for the testnet
- Root or a sudo user
- Go 1.23 or newer (the build may download a newer toolchain automatically, see step 5)
- `curl wget jq git make gcc tar lz4`
- `cosmovisor` (installed in step 4 if missing)
- A Discord account that passes the server's anti-bot captcha
- A free TCP port for P2P (examples use the `41xxx` range, change it if needed)

## 3. Variables

Set these once per terminal session. If you reconnect, export them again.

```bash
export MONIKER="<your-validator-name>"
export WALLET="<your-key-name>"
export P=41                         # port prefix, gives 41656, 41657, ...
export CHAIN_ID=latanda-testnet-1
export CFG=$HOME/.latanda/config
```

The `P` prefix matters if the machine already runs other Cosmos nodes. The default ports 26656 and 26657 are often taken. If you run only this node you can keep the defaults and skip the port edits in step 8.

## 4. Pre-flight checks

```bash
# the output must be empty, otherwise pick another prefix
ss -tlnp | grep -E ":${P}[0-9]{3} "

go version
which cosmovisor || go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

for t in curl wget jq git make gcc tar lz4; do command -v $t >/dev/null || echo "missing: $t"; done
```

If a tool is missing, install only that tool with `apt install <package>`.

## 5. Build the binary

```bash
mkdir -p $HOME/latanda-build && cd $HOME/latanda-build
wget -q https://latanda.online/chain/latanda-chain-source.tar.gz
tar -xzf latanda-chain-source.tar.gz
grep -E '^(go|toolchain) ' go.mod
go mod tidy
go build -o latandad ./cmd/latandad
sha256sum latanda-chain-source.tar.gz
```

Notes:

- The chain is built from the project's own source tarball. There is no release checksum to verify against. Compare the tarball's SHA-256 with other operators in Discord, and keep in mind this proves you downloaded the same file, not that the code is safe.
- If `go.mod` asks for a newer Go than you have installed and your Go is 1.21 or newer, the toolchain is downloaded automatically for this build only. The global Go install is untouched. (In our run, Go 1.25.6 was fetched this way.)
- `latandad version --long` prints empty `version` and `commit` fields. That is expected for this build.

## 6. Cosmovisor layout

```bash
mkdir -p $HOME/.latanda/cosmovisor/genesis/bin $HOME/.latanda/cosmovisor/upgrades
cp $HOME/latanda-build/latandad $HOME/.latanda/cosmovisor/genesis/bin/
ln -sfn $HOME/.latanda/cosmovisor/genesis $HOME/.latanda/cosmovisor/current
ln -sfn $HOME/.latanda/cosmovisor/current/bin/latandad /usr/local/bin/latandad
latandad version
```

The last symlink makes the `latandad` command follow whatever binary Cosmovisor currently uses, so the CLI always matches the running node after an upgrade.

## 7. Init and genesis

```bash
latandad init "$MONIKER" --chain-id $CHAIN_ID --default-denom ultd

wget -O $CFG/genesis.json "https://latanda.online/chain/genesis.json"
sha256sum $CFG/genesis.json
# expected: 98fc9871d6a3b7b12b3f7fcaa1ca3303ffcfad0f209d61355975a15069ac3907

wget -O $CFG/addrbook.json "https://share101.utsa.tech/latanda/addrbook.json"
```

**Stop if the genesis hash does not match.** Ask in Discord for the current hash before continuing.

## 8. Configuration

### client.toml

```bash
sed -i -e "s|^chain-id *=.*|chain-id = \"$CHAIN_ID\"|" \
       -e "s|^keyring-backend *=.*|keyring-backend = \"file\"|" \
       -e "s|^node *=.*|node = \"tcp://localhost:${P}657\"|" $CFG/client.toml
```

The `node` line is important. The default points at port 26657, which on a multi-node machine is probably another chain's RPC, and your CLI commands would silently talk to it.

### app.toml

```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.001ultd\"|" \
       -e "s|^pruning *=.*|pruning = \"custom\"|" \
       -e "s|^pruning-keep-recent *=.*|pruning-keep-recent = \"1000\"|" \
       -e "s|^pruning-interval *=.*|pruning-interval = \"10\"|" \
       -e "s|^address = \"tcp://localhost:1317\"|address = \"tcp://localhost:${P}317\"|" \
       -e "s|^address = \"localhost:9090\"|address = \"localhost:${P}090\"|" $CFG/app.toml
```

### config.toml

```bash
sed -i -e "s|^proxy_app = .*|proxy_app = \"tcp://127.0.0.1:${P}658\"|" \
       -e "s|^laddr = \"tcp://127.0.0.1:26657\"|laddr = \"tcp://127.0.0.1:${P}657\"|" \
       -e "s|^laddr = \"tcp://0.0.0.0:26656\"|laddr = \"tcp://0.0.0.0:${P}656\"|" \
       -e "s|^pprof_laddr = .*|pprof_laddr = \"localhost:${P}060\"|" \
       -e "s|^prometheus_listen_addr = .*|prometheus_listen_addr = \":${P}660\"|" \
       -e "s|^external_address *=.*|external_address = \"$(curl -4 -s https://api.ipify.org):${P}656\"|" \
       -e "s|^persistent_peers *=.*|persistent_peers = \"483a8110c3cd93c8dd3801d935151e98656f5b67@168.231.67.201:26656,d6217d85d3747ebbfbf75898bd4407da567f5291@65.108.206.118:60556\"|" $CFG/config.toml
```

The upstream guide sets `persistent_peers` in two separate places and the second overwrites the first. The line above merges both peers.

Verify the result:

```bash
grep -nE '^(proxy_app|pprof_laddr|prometheus_listen_addr|external_address|persistent_peers)|^laddr' $CFG/config.toml
grep -nE '^node|^keyring-backend|^chain-id' $CFG/client.toml
grep -nE '^(minimum-gas-prices|pruning|pruning-keep-recent|pruning-interval) ' $CFG/app.toml
grep -nE '^address' $CFG/app.toml
```

The ports in `persistent_peers` are the remote peers' own ports and stay as they are. If you opened a firewall, allow `${P}656/tcp`.

## 9. State sync

The old snapshot link in the upstream guide is stale. Use state sync instead.

```bash
SNAP_RPC=https://t-latanda.rpc.utsa.tech:443
LATEST=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
TRUST_H=$((LATEST - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$TRUST_H" | jq -r .result.block_id.hash)
echo $LATEST $TRUST_H $TRUST_HASH     # all three values must be filled, no "null"

# only the state-sync peer while syncing
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"d6217d85d3747ebbfbf75898bd4407da567f5291@65.108.206.118:60556\"|" $CFG/config.toml

sed -i -E -e "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true|" \
          -e "s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"|" \
          -e "s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$TRUST_H|" \
          -e "s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" \
          -e "s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $CFG/config.toml

grep -nE '^(enable|rpc_servers|trust_height|trust_hash|trust_period|persistent_peers) ' $CFG/config.toml
```

`enable = true` must appear only once, under `[statesync]`. Start the service (step 10) and watch the logs. You should see a discovered snapshot, `Applied snapshot chunk` lines and finally `Snapshot restored`.

**Our first attempt failed.** With a trust height only 100 blocks below the tip and the full peer list, state sync discovered old snapshots and the light client kept timing out with `context deadline exceeded`. Using a trust height about 2000 blocks below the tip and a single state-sync peer worked. If you still see timeouts, run the block again, the public RPC can be slow for a moment.

After the node is synced (`catching_up: false`):

```bash
systemctl stop latandad
sed -i -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1false|" $CFG/config.toml
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"483a8110c3cd93c8dd3801d935151e98656f5b67@168.231.67.201:26656,d6217d85d3747ebbfbf75898bd4407da567f5291@65.108.206.118:60556\"|" $CFG/config.toml
systemctl start latandad
```

## 10. systemd service

```bash
tee /etc/systemd/system/latandad.service > /dev/null <<EOF
[Unit]
Description=La Tanda node (cosmovisor)
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $HOME/.latanda
Restart=on-failure
RestartSec=3
LimitNOFILE=65535
Environment="DAEMON_NAME=latandad"
Environment="DAEMON_HOME=$HOME/.latanda"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF

grep -E 'ExecStart|User' /etc/systemd/system/latandad.service   # ExecStart path must not be empty
systemctl daemon-reload && systemctl enable latandad && systemctl start latandad
journalctl -u latandad -f -o cat
```

`UNSAFE_SKIP_BACKUP=true` skips the data directory copy during upgrades. Remove it if you prefer the extra safety and have the disk space. Use `journalctl ... --no-pager` when pasting logs, so the pager does not swallow the next command.

Check sync status:

```bash
curl -s localhost:${P}657/status | jq '.result.sync_info | {latest_block_height, catching_up}'
```

Continue only when `catching_up` is `false`.

## 11. Wallet and faucet

Create a **new** wallet for the validator:

```bash
latandad keys add $WALLET
latandad keys show $WALLET -a      # your ltd1... address
```

The `file` keyring backend asks for a passphrase, keep it. Write the mnemonic down offline.

Importing an existing wallet is possible with `latandad keys add $WALLET --recover`, but we recommend against importing a wallet that holds real funds. The binary is built from a source tarball nobody here audited, and a testnet wallet is all you need. If the address you get differs from the one in your other Cosmos wallet, the chain simply uses a different coin type, that is fine.

**Faucet.** According to the project, the faucet gives 10 LTD, once per person, through the Discord `#node-onboarding` channel after you pass the captcha. Use the exact request format pinned in that channel. Double-check the address before sending it, you only get one drip.

```bash
latandad q bank balances $(latandad keys show $WALLET -a)
```

Expected after the faucet: `10000000ultd`.

## 12. Create the validator

Wait until the node is synced and the balance has arrived.

```bash
cat > $HOME/.latanda/validator.json <<EOF
{
  "pubkey": $(latandad comet show-validator),
  "amount": "1000000ultd",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "<short description>",
  "commission-rate": "0.1",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF

grep '"amount"' $HOME/.latanda/validator.json

latandad tx staking create-validator $HOME/.latanda/validator.json \
  --from $WALLET --chain-id $CHAIN_ID --fees 5000ultd -y
```

The amount is in `ultd`. `1000000ultd` is 1 LTD. The balance must cover the amount plus the fee.

### Do not trust the local output

The command prints a result immediately. A `code: 0` with `height: 0` and `gas_used: 0` is only the local broadcast response, not confirmation from the chain. Always check by hash:

```bash
latandad query tx --type=hash <TX_HASH> --output json | jq '{code, height, raw_log}'
```

The transaction succeeded only if `code` is `0` and `height` is greater than 0. In our case the first attempt printed `code: 0` locally, but the chain result was `code: 5` with `insufficient funds`, because the amount was written with 18 decimals. The failed transaction still burned the 5000ultd fee. If this happens, fix the amount and send again:

```bash
sed -i 's/"amount": "[^"]*"/"amount": "1000000ultd"/' $HOME/.latanda/validator.json
grep '"amount"' $HOME/.latanda/validator.json
```

Confirm the validator exists and is bonded:

```bash
latandad q staking validator $(latandad keys show $WALLET --bech val -a)
latandad q slashing signing-info $(latandad comet show-validator)
```

You want `status: BOND_STATUS_BONDED` and `jailed: false`.

## 13. Discord verification

The project's Discord bot checks two things, a key-control proof (`!verify`) and an infrastructure proof (HTTP-nonce or reverse DNS). The exact steps, memos and nonces are given to you by the bot, the values below are placeholders.

### 13.1 Key control (`!verify`)

The bot gives you a memo. Send exactly 1 `ultd` from your validator wallet to your own address (a self-send) with that memo:

```bash
ADDR=$(latandad keys show $WALLET -a)
latandad tx bank send $WALLET $ADDR 1ultd \
  --note "<MEMO_FROM_BOT>" \
  --chain-id $CHAIN_ID --fees 5000ultd -y
```

The bot's instructions we saw did not include the exact CLI command, this is the standard Cosmos SDK form. Adjust it if the bot says otherwise. Check the transaction by hash as in step 12, then post it in the verification channel:

```
!verify <TX_HASH>
```

The memo must match exactly, character for character.

### 13.2 Infrastructure proof (HTTP-nonce)

The bot asks for either a reverse DNS record or an HTTP nonce. The HTTP route needs the nonce to be reachable on port 80 of your server's public IP.

Check what uses port 80 first:

```bash
curl -4 -s https://api.ipify.org; echo
ss -ltnp | grep ':80 '
```

If a web server such as Nginx already listens on port 80, **do not stop it**. Put the token into its web root instead:

```bash
mkdir -p /var/www/html/.well-known
printf '%s' '<NONCE_FROM_BOT>' > /var/www/html/.well-known/latanda-verify.txt

curl -i http://127.0.0.1/.well-known/latanda-verify.txt
curl -i http://<YOUR_SERVER_IP>/.well-known/latanda-verify.txt
```

Both requests must return `HTTP/1.1 200 OK` and the exact token, with no extra newline (`printf '%s'` takes care of that). Then run the check in Discord:

```
!verify-http-check
```

Use the path and file name the bot gives you, the ones above are what we were asked for. Leave the file in place unless the bot tells you to remove it. If nothing listens on port 80 and you do not have a web server, a temporary minimal HTTP server could serve the file, we did not test that. The reverse DNS alternative needs a PTR record from your hosting provider, also untested here.

### 13.3 Tier checklist

The bot reports a checklist like this. This is what we saw in October 2026 and the program may change:

| Condition | Meaning |
| --- | --- |
| Validator active, bonded, not jailed | `create-validator` succeeded and the node signs |
| `!verify` (key control) | self-send with the bot's memo |
| Infrastructure proof | HTTP-nonce or rDNS |
| 7 days of signing without jail | it showed `0/7` right after setup, so expect to wait |
| Tier delegation | evaluated after the first four conditions |

Per the team's forum post, support delegations go out in batches signed by the team, nothing is automatic, and testnet LTD has no monetary value.

## 14. Monitoring

A running server is not enough. The validator must actually sign blocks.

```bash
# node health
curl -s localhost:${P}657/status | jq '.result.sync_info | {latest_block_height, catching_up}'

# validator state
latandad q staking validator $(latandad keys show $WALLET --bech val -a) | grep -E 'status|jailed|tokens'

# signing and jail info
latandad q slashing signing-info $(latandad comet show-validator)

# logs
journalctl -u latandad -n 50 --no-pager -o cat
```

Watch `missed_blocks_counter` and `jailed_until`. A governance proposal raised the `signed_blocks_window` from 100 to 10,000 during the bootstrap phase, so short outages are less likely to jail you, but a long one still can. If you are jailed, fix the node first, then:

```bash
latandad tx slashing unjail --from $WALLET --chain-id $CHAIN_ID --fees 5000ultd -y
```

Keep the node running without long gaps. The tier condition is 7 days without jail, so a jail will at least delay it.

## 15. Upgrades with Cosmovisor

Build the new binary and put it in an upgrade folder:

```bash
cd $HOME/latanda-build
wget -q https://latanda.online/chain/latanda-chain-source.tar.gz && tar -xzf latanda-chain-source.tar.gz
go build -o latandad ./cmd/latandad
mkdir -p $HOME/.latanda/cosmovisor/upgrades/<UPGRADE_NAME>/bin
cp latandad $HOME/.latanda/cosmovisor/upgrades/<UPGRADE_NAME>/bin/
```

Then one of two cases:

- **The upgrade is an on-chain upgrade plan.** `<UPGRADE_NAME>` must match the plan name exactly. Cosmovisor switches at the upgrade height by itself.
- **The network halts at a block and operators swap the binary by hand.** Switch manually:

```bash
systemctl stop latandad
ln -sfn $HOME/.latanda/cosmovisor/upgrades/<UPGRADE_NAME> $HOME/.latanda/cosmovisor/current
systemctl start latandad
```

We do not know which of the two the team uses. `latandad q upgrade plan` tells you whether the upgrade module is active, and the team's technical channel can confirm how upgrades are coordinated.

## 16. Backups and security

```bash
mkdir -p $HOME/latanda-backup
cp $CFG/priv_validator_key.json $CFG/node_key.json $HOME/latanda-backup/
```

Move these files and your wallet mnemonic **off the server**, encrypted. If the server disk fails and you have no copy, the validator identity is gone.

- Never run the same `priv_validator_key.json` on two machines at once, that is a double sign.
- If you restore from a snapshot on an existing validator, keep your own `priv_validator_state.json`.
- Never post a seed phrase or private key in Discord or to any assistant or bot.
- Use a dedicated wallet for the testnet.
- Do not stop your existing web service just to pass the HTTP check.

## 17. Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `create-validator` prints `code: 0` but the validator does not exist | The output was local only. Query the tx by hash. Check `code` and `height`. |
| `insufficient funds` on-chain | Amount was written with 18 decimals. Use `1000000ultd` (1 LTD). |
| CLI queries return another chain's data | `client.toml` `node` still points to 26657. Set it to `tcp://localhost:${P}657`. |
| State sync: `context deadline exceeded`, old snapshots | Use a trust height about 2000 blocks below the tip, a single state-sync peer, and retry. |
| Genesis hash mismatch | Do not continue. Ask in Discord for the current hash. |
| `ss` shows your ports in use | Pick another `P` prefix and redo step 8. |
| `ExecStart` empty in the service file | `cosmovisor` is not in PATH. Delete the file and recreate it after `which cosmovisor` returns a path. |
| Discord invite fails or the account is removed | The server's anti-bot filter removes accounts that fail the captcha, and the team has said it does not whitelist by hand. |
| `!verify` fails | Memo not exact, wrong tx hash, or the tx did not land. Check it by hash. |
| `!verify-http-check` fails | Port 80 not reachable from the internet, wrong file path, extra newline in the token, or a firewall. Test with `curl` against the public IP. |
| Tier shows `0/7` right after setup | Expected. Keep the validator bonded, signing and out of jail, and check back later. |

## 18. Credits

- UTSA community guide for La Tanda (base steps, addrbook, state sync endpoint)
- La Tanda team forum posts (chain details, support program, tier rules)
- Everyone who tested and reported the problems listed above

Pull requests and corrections are welcome.
