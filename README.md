# Setup NEAR validator node

## 1. Iptables
```shell
iptables -A INPUT -m state --state INVALID -j DROP
iptables -A INPUT -p all -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp --dport 24567 -j ACCEPT
iptables -A INPUT -p tcp --dport $SSH_PORT -j ACCEPT
iptables -P INPUT DROP
iptables -A FORWARD -m state --state INVALID -j DROP
iptables -A FORWARD -p all -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -I FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
iptables -P FORWARD DROP

mkdir /etc/iptables-conf/
touch /etc/iptables-conf/iptables_rules
iptables-save -f /etc/iptables-conf/iptables_rules
nano /etc/network/if-pre-up.d/iptables

#!/bin/sh
iptables-restore < /etc/iptables-conf/iptables_rules

chmod +x /etc/network/if-pre-up.d/iptables
```

## 2. SSH
```shell
nano /etc/ssh/sshd_config

Port $SSH_PORT
Protocol 2
PermitRootLogin no
AllowUsers nearnodeuser

systemctl restart ssh
```

## 3. Install dependencies
```shell
apt -y update && apt -y upgrade
apt -y install git curl sudo make clang net-tools cargo wget ccze
# apt -y install cmake g++ pkg-config libssl-dev llvm
# The RUST
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## 4. User
```shell
adduser nearnodeuser
```
  
## 5. Environment
```shell
export NEAR_RELEASE_VERSION=$(curl -s https://github.com/near/nearcore/releases/latest | tr '/" ' '\n' | grep "[0-9]\.[0-9]*\.[0-9]" | head -n 1)
export NEAR_ENV=mainnet
export NODE_ENV=mainnet
```
or
```shell
nano /home/nearnodeuser/.bashrc

export NEAR_ENV=mainnet
export NODE_ENV=mainnet
```

## 6. Copy the blockchain
```shell
wget -O /home/nearnodeuser/.near/data.tar https://near-protocol-public.s3.ca-central-1.amazonaws.com/backups/mainnet/rpc/data.tar
mkdir -p /home/nearnodeuser/.near/data
tar -xf /home/nearnodeuser/.near/data.tar -C /home/nearnodeuser/.near/data
rm /home/nearnodeuser/.near/data.tar
```

## 7. Copy the config
```shell
rm /home/nearnodeuser/.near/config.json
wget -O /home/nearnodeuser/.near/config.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/config.json
```

## 8. Run the node
```shell
git clone https://github.com/near/nearcore.git
cd nearcore
git checkout $NEAR_RELEASE_VERSION
make neard
target/release/neard init --chain-id="mainnet" --account-id=cryptostore
target/release/neard run
```

## 9. Make autorun
```shell
# Only after copy the neard.service to /etc/systemd/system
systemctl start neard.service
systemctl status neard.service
systemctl enable neard.service
```

## 10. The status of the node
```shell
# node version
curl -s http://127.0.0.1:3030/status | jq .version
# log
journalctl -n 100 -f -u neard | ccze -A
```

## 11. Host which a wallet
```shell
curl -sL https://deb.nodesource.com/setup_17.x | bash -
apt -y update && apt -y install nodejs
apt install nodejs
npm install -g near-cli


near call poolv1.near create_staking_pool '{"staking_pool_id":"cryptostore", "owner_id":"cryptostore.near", "stake_public_key":"ed25519:GCZgigwcDxJRaZnhyDyW1uRq1aZFMyKRcqaPtxAaj3LL", "reward_fee_fraction": {"numerator": 5, "denominator": 100}}' --account_id <ownwer_wallet_id>.near --amount 30 --gas 300000000000000
near call name.near update_field '{"pool_id": "cryptostore.poolv1.near", "name": "email", "value": "aleksandr.sakh@gmail.com"}' --accountId=webdev.near  --gas=200000000000000
near call name.near update_field '{"pool_id": "cryptostore.poolv1.near", "name": "country", "value": "russia"}' --accountId=webdev.near  --gas=200000000000000
```

### 12. For reset your credentials
```shell
target/release/neard unsafe_reset_all
```
