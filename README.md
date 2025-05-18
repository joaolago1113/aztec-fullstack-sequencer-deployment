# Aztec Sequencer – Local EL & CL (Geth + Prysm + Aztec)

 **Sanitised** – replace **ALL** placeholders **before running**:
 
   • `YOUR_PUBLIC_IP`
   
   • `0xYOUR_COINBASE_ADDRESS`
   
   • `0xYOUR_VALIDATOR_PRIVATE_KEY`
   
Follow the steps in order; commands are copy‑paste ready for a bare server.

---

## 0  Prerequisites

* 64‑bit Linux
* 800 GB free
* sudo privileges
* Open ports 30303/tcp+udp, 40401/tcp+udp

Define your operator user once per session and export it so that every file we generate is concrete:

```bash
export NODEUSER=$(logname)     
export GETH_VER=1.15.10-2bf8a789
export ARCH=$(uname -m)
case "$ARCH" in
  x86_64|amd64)  GETH_PKG=geth-linux-amd64-$GETH_VER.tar.gz ;;
  aarch64|arm64) GETH_PKG=geth-linux-arm64-$GETH_VER.tar.gz ;;
  armv7l|armv6l) GETH_PKG=geth-linux-arm5-$GETH_VER.tar.gz  ;;
  *) echo "Unsupported architecture $ARCH"; exit 1 ;;
esac
```

---

## 1  Core packages & time sync

```bash
sudo apt update && sudo apt install -y curl gnupg openssl systemd-timesyncd docker.io
# install & launch Docker
sudo apt update && sudo apt install -y docker.io
# start and enable the daemon
sudo systemctl start docker.service
sudo systemctl enable docker.service

# allow the current user to run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# verify the daemon is up
sudo systemctl --no-pager status docker.service

# time sync
sudo apt install -y systemd-timesyncd
sudo systemctl enable --now systemd-timesyncd.service
```

---

## 2  Geth (execution layer)

```bash
cd ~
wget -q https://gethstore.blob.core.windows.net/builds/$GETH_PKG
 tar -xf $GETH_PKG
sudo mv ${GETH_PKG%.tar.gz}/geth /usr/local/bin/
sudo chmod +x /usr/local/bin/geth
```

### systemd unit

```bash
export NODEUSER=$(logname) 
sudo tee /etc/systemd/system/geth.service > /dev/null <<EOF
[Unit]
Description=Geth Sepolia Node
After=network.target

[Service]
User=$NODEUSER
ExecStart=/usr/local/bin/geth \
  --sepolia \
  --datadir /home/$NODEUSER/ethereum/data/geth \
  --syncmode snap \
  --http --http.addr 0.0.0.0 --http.port 8545 \
  --http.api eth,net,web3,engine,admin,debug \
  --http.corsdomain "*" \
  --http.vhosts localhost,127.0.0.1,host.docker.internal,172.17.0.1 \
  --authrpc.addr 0.0.0.0 --authrpc.port 8551 \
  --authrpc.vhosts localhost \
  --authrpc.jwtsecret /home/$NODEUSER/ethereum/data/jwtsecret
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

---

## 3  JWT secret

```bash
sudo -u $NODEUSER bash -c 'mkdir -p ~/ethereum/data && openssl rand -hex 32 | tr -d "\n" > ~/ethereum/data/jwtsecret && chmod 400 ~/ethereum/data/jwtsecret'
```

---

## 4  Prysm (consensus layer)

```bash
export NODEUSER=$(logname)
sudo install -d -o "$NODEUSER"  /opt/prysm
sudo -u "$NODEUSER" curl -L https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh -o /opt/prysm/prysm.sh
sudo chmod +x /opt/prysm/prysm.sh
```

### systemd unit

```bash
sudo tee /etc/systemd/system/prysm-beacon.service > /dev/null <<EOF
[Unit]
Description=Prysm – Sepolia beacon‑chain
Wants=geth.service network-online.target
After=geth.service network-online.target

[Service]
User=$NODEUSER
WorkingDirectory=/opt/prysm

ExecStart=/opt/prysm/prysm.sh beacon-chain \
  --sepolia \
  --execution-endpoint=http://127.0.0.1:8551 \
  --jwt-secret=/home/$NODEUSER/ethereum/data/jwtsecret \
  --checkpoint-sync-url=https://sepolia.beaconstate.info \
  --genesis-beacon-api-url=https://sepolia.beaconstate.info \
  --enable-experimental-backfill \
  --http-host=0.0.0.0 \
  --accept-terms-of-use
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

Reload and (re)start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prysm-beacon.service
```

---

## 5  Run Aztec Sequencer

At this point you can run the node as usual, but replace the RPC URLs with the following:

```
export YOUR_RPC_ENDPOINT=http://localhost:8545
export YOUR_CONSENSUS_ENDPOINT=http://localhost:3500
````

If you want a more robust setup for your node, which boots up on reboot of your machine then continue copy-pasting:

### Binary install

```bash
# Ensure NODEUSER is defined (fallback to current login user)
NODEUSER=${NODEUSER:-$(logname)}

sudo -u "$NODEUSER" bash -i <(curl -s https://install.aztec.network)
# Fetch the latest binaries for the public alpha testnet
sudo -u "$NODEUSER" bash -c 'echo "export PATH=\$HOME/.aztec/bin:\$PATH" >> ~/.bash_profile && source ~/.bash_profile'
export PATH="$HOME/.aztec/bin:$PATH"
sudo -u "$NODEUSER" aztec-up alpha-testnet

```

### Validator key

```bash
sudo mkdir -p /etc/aztec
echo '0xYOUR_VALIDATOR_PRIVATE_KEY' | sudo tee /etc/aztec/validator.key > /dev/null
sudo chmod 600 /etc/aztec/validator.key
```

### Launch wrapper

```bash
sudo tee /usr/local/bin/run-aztec.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

export COINBASE=0xYOUR_COINBASE_ADDRESS
export DATA_DIRECTORY=$HOME/aztec-data
export LOG_LEVEL=debug
export YOUR_IP_ADDRESS=YOUR_PUBLIC_IP
export YOUR_RPC_ENDPOINT=http://localhost:8545
export YOUR_CONSENSUS_ENDPOINT=http://localhost:3500
export OTEL_COLLECTOR_ENDPOINT="http://host.docker.internal:4318"
export P2P_MAX_TX_POOL_SIZE=1000000000

YOUR_VALIDATOR_PRIVATE_KEY=$(cat /run/credentials/aztec.service/validator.key)

cd $HOME

$HOME/.aztec/bin/aztec start \
  --network alpha-testnet \
  --l1-rpc-urls "$YOUR_RPC_ENDPOINT" \
  --l1-consensus-host-urls "$YOUR_CONSENSUS_ENDPOINT" \
  --sequencer.validatorPrivateKey "$YOUR_VALIDATOR_PRIVATE_KEY" \
  --p2p.p2pIp "$YOUR_IP_ADDRESS" \
  --p2p.p2pPort 40401 \
  --port 8081 \
  --archiver --node --sequencer
EOF

sudo chmod 755 /usr/local/bin/run-aztec.sh
```

### systemd unit

```bash
export NODEUSER=$(logname) 
sudo tee /etc/systemd/system/aztec.service > /dev/null <<EOF
[Unit]
Description=Aztec Node
After=prysm-beacon.service
Wants=prysm-beacon.service

[Service]
User=$NODEUSER
Type=simple
Environment=HOME=/home/$NODEUSER
ExecStart=/usr/local/bin/run-aztec.sh
Restart=always
RestartSec=5
LoadCredential=validator.key:/etc/aztec/validator.key

[Install]
WantedBy=multi-user.target
EOF
```

---

## 6  Activation

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now geth.service prysm-beacon.service aztec.service
```

---

## 7  Monitoring

```bash
systemctl status geth.service prysm-beacon.service aztec.service
journalctl -u geth.service -f
journalctl -u prysm-beacon.service -f
journalctl -u aztec.service -f
```

Follow this guide to monitor your node properly using grafana: `https://hackmd.io/@aztec-network/BJjKGUP3ye`

---

Stack ready. Re‑run activation after any edits:

```bash
sudo systemctl daemon-reload && sudo systemctl restart geth prysm-beacon aztec
```
