# Thornode – Linux

Setting up a fullnode on Linux

The steps shown here are tested on `Ubuntu 24.04`; different distributions may need adjustments to the commands.

All commands are meant to be run as root user unless specified otherwise. Depending on the server installation, they may need to be run from a different user via `sudo`.

## Prerequisites

Install all needed packages for building and configuring the THORNode daemon:

```bash
sudo apt install -y --no-install-recommends aria2 ca-certificates curl git golang jq make pv gawk build-essential libssl-dev pkg-config libclang-dev cmake rustc cargo
```

## Application user

Add the application user that is used to run the THORNode application and switch to it:

```bash
sudo useradd -m thornode -s /bin/bash
sudo su - thornode
```

## Build
Checkout the latest code and build the binary.

Download the THORNode source code at release tag v3.8.0 into a new folder called build inside the thornode user’s 
home directory (/home/thornode/build).  

Using --branch pins you to that exact version instead of whatever is on main.

```bash
git clone --branch v3.8.0 https://gitlab.com/thorchain/thornode $HOME/build
cd $HOME/build
```

NO LONGER NEEDED WITH CGO?
The THORNode build process requires a specific version of the CosmWasm/wasmd module:
```
go mod edit -replace github.com/CosmWasm/wasmd=github.com/CosmWasm/wasmd@v0.53.4
go mod tidy
```
Docker work-around:
```bash
ln -fs /usr/bin/true docker; export PATH=$(pwd):$PATH; CGO_ENABLED=1 TAG=mainnet make install
# remove bifrost binary and build directory
rm $HOME/go/bin/bifrost
cd
rm -rf $HOME/build
```

> **Note**  
> THORNode’s Makefile checks for a docker binary even though Docker isn’t required to build the Go code. This link satisfies the check without pulling in Docker.

`export PATH=$(pwd):$PATH`: Prepend the current directory to PATH so the newly-created fake docker (and any other 
helper scripts in the repo) are found before system binaries during the build.

`TAG=mainnet make install`: Invoke the Makefile’s install target with TAG=mainnet. The variable stamps the binary with the “mainnet” build tag and places the compiled thornode executable into $HOME/go/bin/.

`rm $HOME/go/bin/bifrost`: Remove bifrost binary (Bifrost is the signer-bridge used by validator nodes to talk to 
their Yggdrasil wallets; a 
“full-node / observer” machine doesn’t need it):

`rm -rf $HOME/build`: Remove build directory:

## Prepare environment

### Config

Create the configuration files and directory layout:  

```bash
$HOME/go/bin/thornode init thornode --overwrite --chain-id thorchain-1
```

**Initialize the node’s data directory**

| Flag / Argument | Explanation                                                                                                                                           |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `init` | Runs THORNode’s one-time setup routine, creating `$HOME/.thornode/` with default configs (app.toml, config.toml, genesis.json) and an empty database. |
| `thornode` | **Moniker** – the human-readable name your node advertises to peers and explorers.                                                                    |
| `--overwrite` | If `$HOME/.thornode` already exists, wipe it and re-initialize (handy when starting over).                                                            |
| `--chain-id thorchain-1` | Locks the config to THORChain **main-net** so the node won’t join a testnet or fork.                                                                  |

### Seed nodes

Seeds provide a list of active THORChain nodes, which are needed to join the network. Replace the empty **seed list** in `config.toml` with two reliable Ninerealms seed nodes:

```bash
sed -i 's/^seeds = ""/seeds = "c3613862c2608b3e861406ad02146f41cf5124e6@statesync-seed.ninerealms.com:27146,dbd1730bff1e8a21aad93bc6083209904d483185@statesync-seed-2.ninerealms.com:27146"/'   $HOME/.thornode/config/config.toml
```

### Ports

THORChain doesn't use the Cosmos‑SDK default ports. Technically this step isn't needed, but it is meant to stay in line with all other THORNode deployments.

Update all default Cosmos-SDK **port numbers** in `config.toml` to the THORChain-specific range:

```bash
sed -ri 's/:2665([0-9])/:2714\1/g' $HOME/.thornode/config/config.toml
```

### Genesis
For joining the network, the correct genesis file is required.

Fetch the **main-net genesis file** and save it in the expected location:

```bash
curl https://storage.googleapis.com/public-snapshots-ninerealms/genesis/17562000.json -o $HOME/.thornode/config/genesis.json
```
>**Note**
> -o …/genesis.json overwrites the placeholder created by thornode init, ensuring your node starts with the correct initial state.
### Sync
The fastest way to join the network is by downloading a current snapshot and syncing from it.

Query Ninerealms, filter, and pick the highest block-height file name (e.g., 20840000.tar.gz). Then, store the 
result in the shell variable FILENAME:
```bash

FILENAME=$(curl -s "https://snapshots.ninerealms.com/snapshots?prefix=thornode" \
  | grep -Eo "thornode/[0-9]+.tar.gz" \
  | sort -n \
  | tail -n 1 \
  | cut -d "/" -f 2)
```
* **`curl -s …`** – fetch the raw object listing (quiet mode).  
* **`grep -Eo 'thornode/[0-9]+\.tar\.gz'`** – keep only file names like `thornode/<height>.tar.gz`.  
* **`sort -n`** – sort those names numerically by height.  
* **`tail -n 1`** – take the last entry → the **newest snapshot**.  
* **`cut -d "/" -f 2`** – drop the `thornode/` prefix, yielding just `12345678.tar.gz`.


Download the chosen snapshot with **aria2c** (fast, resume-capable):
```bash
aria2c --split=16 --max-concurrent-downloads=16 --max-connection-per-server=16 \
  --continue --min-split-size=100M \
  -d $HOME/.thornode -o $FILENAME \
  "https://snapshots.ninerealms.com/snapshots/thornode/${FILENAME}"
```
* **`--split=16` / `--max-concurrent-downloads=16`** – split the file into 16 parts and download them in parallel, maximizing throughput.  
* **`--max-connection-per-server=16`** – let all 16 parts be fetched simultaneously from the same host.  
* **`--continue`** – resume an interrupted download instead of starting over.  
* **`--min-split-size=100M`** – only split files ≥ 100 MB (the snapshot is ~5 GB, so splitting applies).  
* **`-d "$HOME/.thornode"`** – save the file directly in the THORNode directory.  
* **`-o "$FILENAME"`** – write the file using the name captured earlier.  
* **URL argument** – the full Ninerealms snapshot path built with `${FILENAME}`.

Remove any **stale chain data** before you unpack the fresh snapshot:

```bash
rm -rf $HOME/.thornode/data/{*.db,snapshot,cs.wal}
```

* **`*.db`** – deletes existing LevelDB databases (`application.db`, `state.db`, `evidence.db`, …)  
* **`snapshot/`** – removes any incomplete or outdated snapshot directory  
* **`cs.wal`** – erases the consensus write-ahead log to avoid replay conflicts


Stream-extract the snapshot with a progress bar (leave out `--exclude "*_state.json` flag to prevent syncing error):

```
pv $HOME/.thornode/$FILENAME | tar -xzf - -C $HOME/.thornode
```

* **`pv …`** – streams the file through *Pipe Viewer*, showing real-time progress and speed.  
* **`tar -xzf -`** – reads the snapshot from standard input (`f -`), un-gzips (`z`) and extracts (`x`) it.  
* **`-C "$HOME/.thornode"`** – switches to the THORNode directory first, so all extracted databases and metadata land in the proper place.


Delete the now-unneeded snapshot archive to free disk space:

```bash
rm -rf $HOME/.thornode/$FILENAME
```

Set a non-zero minimum gas price:

`nano $HOME/.thornode/config/app.toml`

Change `minimum-gas-prices = ""` to `minimum-gas-prices = "0.03rune"`

## Edit $HOME.thornode/config/config.toml

Under `[statesync]` set: 
`enable = true`
`rpc_servers  = "https://rpc.ninerealms.com:443,https://thornode.ninerealms.com:443"`
`trust_hash = "4D1B2804A6796B7930AABFD5E8573980A7F8CC79269907C6DB29449918A05109"`

Become primary user:
`exit`

## Systemd [maybe this file need revision...]

Create a service file so you can manage the Thornode process via systemd:

`sudo nano /etc/systemd/system/thornode.service`

Paste:
```ini
[Unit]
Description=Thornode Daemon
After=network-online.target

[Service]
User=thornode
Environment=HOME=/home/thornode
ExecStart=/home/thornode/go/bin/thornode start  \
          --home /home/thornode/.thornode
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```
> **Tip**  
>For automatic restarts when the RPC freezes, add a thornode-health.timer that hits http://127.0.0.1:27147/status every few minutes.

Reload systemd configuration:

`sudo systemctl daemon-reload`

Enable service:

`sudo systemctl enable thornode`

Start service:

`sudo systemctl start thornode.service`

## Configure UFW firewall
### Install ufw if not present
`sudo apt install -y ufw`

### Default policy:
Deny unsolicited inbound

`sudo ufw default deny incoming`

Allow all outbound

`sudo ufw default allow outgoing`

### Open THORNode ports
`sudo ufw allow 27146/tcp`          # P2P

`sudo ufw allow 27147/tcp`          # RPC (state-sync, status)

### (Optional) Midgard/API
`sudo ufw allow 8080/tcp`        

### Keep SSH open (usually already allowed on Ubuntu)
`sudo ufw allow 22/tcp`

### Enable the firewall
`sudo ufw enable`

### Verify
`sudo ufw status numbered`

### (Optional) Limit journald log size

THORNode’s verbose Tendermint logs can grow several GB over time.  
Capping the journal prevents a full disk without installing extra packages.

```bash
sudo sed -i 's/^#SystemMaxUse=.*/SystemMaxUse=2G/' /etc/systemd/journald.conf
sudo sed -i 's/^#SystemMaxFileSize=.*/SystemMaxFileSize=200M/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```

* **`SystemMaxUse`**: journald keeps truncating the oldest files until the directory size drops below 2 GB.  
* **`SystemMaxFileSize`**: each live journal file tops out at ~200 MB, giving you about 10 files in the rotation.  
* The restart makes the new limits effective immediately; no need to reboot.

