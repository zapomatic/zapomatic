# PeerSwap on an Umbrel Home

Feeling reckless today, [Zap-O-Matic](https://amboss.space/node/026d0169e8c220d8e789de1e7543f84b9041bbb3e819ab14b9824d37caa94f1eb2) installed [PeerSwap](https://www.peerswap.dev/) and reached out to [FriendsPool](https://amboss.space/node/023e24602891c28a7872ea1ad5c1bb41abe4206ae1599bb981e3278a121e7895d6) to coordinate a swap (it doesn't require explicit coordination but it's a good idea to make sure your node partner will have the L-BTC to send you for a swap out or that they want to take your L-BTC in exchange for lightning sats).

It went without a hitch! And to our surprise, Liquid BTC transactions have confidential UTXO sizes. The only thing you can see in the history is the addresses and the fees: https://liquid.network/ -- Kinda neat.

There's a really helpful blog guide by [Vlad](https://github.com/Impa10r) who is working on an Umbrel App store app for his [PeerSwap Web UI](https://github.com/Impa10r/peerswap-web) for reference: https://medium.com/@goryachev/liquid-rebalancing-of-lightning-channels-2dadf4b2397a

Here's how you can do this on an Umbrel Home with LND:

## Install Elements Core

1. Install Elements Core from the Umbrel App Store

```
echo "trim_headers=1" > /home/umbrel/umbrel/app-data/elements/data/elements.conf
~/umbrel/scripts/app restart elements
```

> Note: your umbrel might be setup to require `sudo` for app/docker commands

2. While Elements syncs (can take a bit and uses a LOT of memory while it does so), we can prep everything else:

## Prep Environment

> Note: you can install and run peerswap on another machine that is not where you are running your LND node if you have copied out your tls.cert and admin.macaroon files from your LND node. You can also run peerswap on the same machine as your LND node. The following instructions are simplified to assume you are runnning it directly on the Umbrel Home.

1. Make sure you have `build-essential` so you can run `make` (ssh into machine and run this):

```
sudo apt update
sudo apt install build-essential
```

2. make sure you have Go installed (PeerSwap is a go app)
3. For an Umbrel Home, get the latest amd64 binary from https://go.dev/dl/
4. run the following from ssh cli:

```
cd
VERSION=1.21.5
ARCH=amd64
wget https://go.dev/dl/go$VERSION.linux-$ARCH.tar.gz
# note: I got this checksum from the release page: https://go.dev/dl/
echo -n "e2bc0b3e4b64111ec117295c088bde5f00eeed1567999ff77bc859d7df70078e *go${VERSION}.linux-${ARCH}.tar.gz" | shasum -a 256 --check

# if the checksum matches, you can extract and install

tar -xf "go${VERSION}.linux-${ARCH}.tar.gz"
sudo mv -v go /usr/local/
```

5. add the following to ~/.profile (vim ~/.profile, or nano or whatever you like to use)

```
export GOPATH=$HOME/go
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin" >> ~/.profile
```

6. resource your profile to get ENV vars updated and verify go is installed:

```
source ~/.profile
go version
```

## Install PeerSwap

Referencing: https://medium.com/@goryachev/liquid-rebalancing-of-lightning-channels-2dadf4b2397a
and: https://github.com/ElementsProject/peerswap/blob/master/docs/setup_lnd.md

```
git clone https://github.com/ElementsProject/peerswap.git && \
cd peerswap && \
make lnd-release

# make should work and it will install pscli and peerswapd in your $GOPATH :)

mkdir -p ~/.peerswap

# create config file (replace password from elements core app)
cat <<EOF > ~/.peerswap/peerswap.conf
lnd.tlscertpath=/home/umbrel/umbrel/app-data/lightning/data/lnd/tls.cert
lnd.macaroonpath=/home/umbrel/umbrel/app-data/lightning/data/lnd/data/chain/bitcoin/mainnet/admin.macaroon
elementsd.rpcuser=elements
elementsd.rpcpass=<REPLACE_ME with password from elements app>
elementsd.rpchost=http://127.0.0.1
elementsd.rpcport=7041
elementsd.rpcwallet=peerswap
elementsd.liquidswaps=true
bitcoinswaps=false
EOF

# setup sysetem deamon
sudo cat <<EOF > /etc/systemd/system/peerswapd.service
[Unit]
Description=Peer Swap Daemon
[Service]
ExecStart=/home/umbrel/go/bin/peerswapd
User=umbrel
Type=simple
KillMode=process
TimeoutSec=180
Restart=always
RestartSec=60
[Install]
WantedBy=multi-user.target
EOF

# start service
sudo systemctl start peerswapd
sudo systemctl status peerswapd
sudo systemctl enable peerswapd
```

## Test and Initiate Swap!

Once Elements has synced, we can run a swap!

`pscli` should be available via your GOPATH. If not, make sure you have your GOPATH set in your shell and do `make lnd-release` again in the peerswap repo to reinstall

```
# get a list of all peers that can use peerswap
# this will return even peers that we do not have a channel to (yet)
pscli listpeers
```

## Add a Peer to Allow Swaps

PeerSwap by default does not allow swap requests. You can open it up to anyone, but this is all still experimental so let's be explicit

```
# allow swaps with Zap-O-Matic
pscli addpeer --peer_pubkey 026d0169e8c220d8e789de1e7543f84b9041bbb3e819ab14b9824d37caa94f1eb2
# allow swaps with FriendsPool
pscli addpeer --peer_pubkey 023e24602891c28a7872ea1ad5c1bb41abe4206ae1599bb981e3278a121e7895d6
```

## Make a Swap Request

```
# get the channel id that we have open to Zap-O-Matic:
# assuming we have opened a channel. If not, why not? :)

CHAN_ID=$(/home/umbrel/umbrel/scripts/app compose lightning exec lnd lncli listchannels --active_only --peer 026d0169e8c220d8e789de1e7543f84b9041bbb3e819ab14b9824d37caa94f1eb2 | grep '"chan_id"' | cut -d':' -f2 | tr -d '", ')

# Attempt a swap out of 1M sats (move our balance to the other side in exchange for LBTC)

pscli swapout --sat_amt=1000000 --channel_id=$CHAN_ID--asset=lbtc

# this will probably timeout since they are not waiting to accept (unless we are chatting live)

# but it will be pending acceptance from the peer and you can view the status of your request

pscli listswaps
```

Once the swap is accepted by your channel partner, it will give you a transaction ID that you can view on the Liquid mempool.space: https://liquid.network/

```
# Once it clears, you will see your balance in your lbtc wallet:

pscli lbtc-getbalance
```

And we have done it!

Now, optionaly, let's tell all of our connected peers that we have PeerSwap working with a 1 sat keysend message and some cli-fu using `bos`:

```
bos peers --no-color --active --complete | grep public_key | awk '{print $2}' | xargs -I {} bash -c 'bos send {} --amount 1 --message "📢 Hello channel partner, we are now using PeerSwap for rebalancing, which is fast, reliable, and cost-effective among my peers. The more peers using it, the more everyone benefits! 🤲 Join us on this endeavor. It will be worth it! Check out a guide here: https://github.com/zapomatic/zapomatic/blob/main/PeerSwap.md"'
```
