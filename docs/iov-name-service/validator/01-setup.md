---
id: setup
title: Setup A Validator Node
sidebar_label: Setup
---

## Apply to the validator program

Before starting to setup your validator, please <a href="https://support.iov.one/hc/en-us/requests/new?ticket_form_id=360000417771" target="_blank">apply to the validator program</a> to open a channel of communication with IOV. At the end of the article, you will need to use this channel of communication to give us your pub_key so that we can upgrade your full-node to a validator.

## Familiarize yourself with Gitian

Downloading and running a binary makes most sane people nervous.  <a href="https://gitian.org/" target="_blank">Gitian</a> introduces a level of trust for binary artefacts and is the <a href="https://medium.com/iov-internet-of-values/distribute-open-source-software-the-right-and-verifiable-way-fe12f58df062" target="_blank">distribution method</a> chosen by IOV and other blockchains including Bitcoin and <a href="https://medium.com/tendermint/reproducible-builds-8c2eebb9a486" target="_blank">Cosmos</a>.  We'll use binaries built using gitian and systemd to drive the IOV Name Service blockchain.

## Use systemd for running a sentry node or validator

This document is not for beginners.  It assumes that you know how to setup a sentry node architecture for Tendermint nodes.

> Hint: When upgrading from an old testnet to a new one, you can maintain your node id and validator pub_key across testnets by doing the following before performing the upgrade:

```sh
su - iov
set -o allexport ; source /etc/systemd/system/iovns.env ; set +o allexport # pick-up env vars
cp -av ${DIR_WORK}/config/*_key.json ~
exit
```

This document assumes that `basename`, `curl`, `expr`, `grep`, `jq`, `sed`, and `wget` are installed on your system, and user `iov` exists.  You should be able to copy-and-paste the following commands into a terminal and end up with a running node.  You'll have to do this procedure on at least two machines to implement a sentry node architecture.

```sh
sudo su # make life easier for the next ~100 lines

cd /etc/systemd/system

# create an environment file for the IOV Name Service services
cat <<__EOF_IOVNS_ENV__ > iovns.env
# directories (without spaces to ease pain)
DIR_IOVNS=/opt/iovns/bin
DIR_WORK=/home/iov/mainnet

# images
IMAGE_IOVNS=https://github.com/iov-one/weave/releases/download/v0.21.2/bnsd-0.21.2-linux-amd64.tar.gz
IMAGE_IOVNS_OPTS=""
IMAGE_TM=https://github.com/iov-one/tendermint-build/releases/download/v0.31.9-iov1/tendermint-0.31.9-linux-amd64.tar.gz
IMAGE_TM_OPTS="\
--consensus.create_empty_blocks=false \
--moniker='moniker' \
--p2p.laddr=tcp://0.0.0.0:16656 \
--p2p.seeds=352ba402a2461020689c86cab599c8a44bd49a33@35.198.191.90:26656,88af4a6c555c058e530c124babfa9f9fb12a01b2@35.234.78.200:26656 \
--rpc.laddr=tcp://127.0.0.1:16657 \
--rpc.unsafe=false \
"

# socket
SOCK_TM=iovns.sock

# uid/gid
IOV_GID=$(id iov -g)
IOV_UID=$(id iov -u)
__EOF_IOVNS_ENV__

chgrp iov iovns.env
chmod g+r iovns.env

set -o allexport ; source /etc/systemd/system/iovns.env ; set +o allexport # pick-up env vars

# create iovns.service
cat <<'__EOF_IOVNS_SERVICE__' | sed -e 's@__DIR_IOVNS__@'"$DIR_IOVNS"'@g' > iovns.service
[Unit]
Description=IOV Name Service
After=network-online.target
Requires=iovns-tm.service
PartOf=iovns-tm.service

[Service]
Type=simple
User=iov
Group=iov
EnvironmentFile=/etc/systemd/system/iovns.env
ExecStart=__DIR_IOVNS__/bnsd \
   -home=${DIR_WORK} \
   start \
   -bind=unix://${DIR_WORK}/${SOCK_TM} \
   $IMAGE_IOVNS_OPTS
LimitNOFILE=4096
Restart=on-failure
RestartSec=3
StandardError=journal
StandardOutput=journal
SyslogIdentifier=iovns

[Install]
WantedBy=multi-user.target
__EOF_IOVNS_SERVICE__

# create iovns-tm.service
cat <<'__EOF_IOVNS_TM_SERVICE__' | sed -e 's@__DIR_IOVNS__@'"$DIR_IOVNS"'@g' > iovns-tm.service
[Unit]
Description=Tendermint for IOV Name Service
After=iovns.service
Requires=iovns.service

[Service]
Type=simple
User=iov
Group=iov
EnvironmentFile=/etc/systemd/system/iovns.env
ExecStart=__DIR_IOVNS__/tendermint node \
   --home=${DIR_WORK} \
   --proxy_app=unix://${DIR_WORK}/${SOCK_TM} \
   $IMAGE_TM_OPTS
LimitNOFILE=4096
Restart=on-failure
RestartSec=3
StandardError=journal
StandardOutput=journal
SyslogIdentifier=iovns-tm

[Install]
WantedBy=multi-user.target iovns.service
__EOF_IOVNS_TM_SERVICE__

# hack around ancient versions of systemd
expr $(systemctl --version | grep -m 1 -P -o "\d+") '<' 239 && {
   sed --in-place 's!\$IMAGE_IOVNS_OPTS!'"$IMAGE_IOVNS_OPTS"'!' /etc/systemd/system/iovns.service
   sed --in-place 's!\$IMAGE_TM_OPTS!\'"$IMAGE_TM_OPTS"'!' /etc/systemd/system/iovns-tm.service
}

systemctl daemon-reload

# download gitian built binaries; bnsd is the IOV Name Service daemon
mkdir -p ${DIR_IOVNS} && cd ${DIR_IOVNS}
wget ${IMAGE_IOVNS} && sha256sum $(basename $IMAGE_IOVNS) | fgrep d1cba6d3a43a555875421d14a6c8d05660a2f1fd51e6f762707520aed9af10fe && tar xvf $(basename $IMAGE_IOVNS) || echo "BAD BINARY!"
wget ${IMAGE_TM}    && sha256sum $(basename $IMAGE_TM) | fgrep a23faa07bc91adb736e7f9df52c48aea262a359fa758862c5ff86740d9c66341 && tar xvf $(basename $IMAGE_TM) || echo "BAD BINARY!"

exit # root

# initialize the IOV Name Service
su - iov
set -o allexport ; source /etc/systemd/system/iovns.env ; set +o allexport # pick-up env vars

mkdir -p ${DIR_WORK} && cd ${DIR_WORK}

# initialize tendermint
${DIR_IOVNS}/tendermint init --home=${DIR_WORK}
curl --fail https://gist.githubusercontent.com/webmaster128/9a87d0967fe2caa95d84ee6288c648c2/raw/70c95107b2b4cb8ed3c0d24ae1f3f43a55d81cff/genesis.json > config/genesis.json
sha256sum config/genesis.json | grep 6c80ea4724726bedd2d36e73bf025007ef898fcb06be17e3ba3e51f32d29b8fa || echo "BAD GENESIS FILE!"
[[ -f ~/node_key.json ]] && cp -av ~/node_key.json config
[[ -f ~/priv_validator_key.json ]] && cp -av ~/priv_validator_key.json config
sed --in-place 's!^timeout_commit .*!timeout_commit = "5s"!' config/config.toml # options not available via command line
sed --in-place 's!^create_empty_blocks .*!create_empty_blocks = false!' config/config.toml
sed --in-place 's!^create_empty_blocks_interval .*!create_empty_blocks_interval = "300s"!' config/config.toml

# initialize IOV Name Service (bnsd)
${DIR_IOVNS}/bnsd -home=${DIR_WORK} init -i | grep initialised

exit # iov

journalctl -f | grep iovns & # watch the chain sync
systemctl start iovns.service
```

At this point you're running a full-node that can be examined at `http://localhost:16657/status`.

> The most important file from the procedure above is `/etc/systemd/system/iovns.env`.  It defines binaries, the socket that allows `iovns.service` and `iovns-tm.service` to communicate with each other, and IOV Name Service and tendermint options.

Using `/etc/systemd/system/iovns.env`, rather than specifying values directly in the service files, obviates the need to do `systemctl daemon-reload` on every option change.  Most values in `/etc/systemd/system/iovns.env` are self explanatory; however, there are a few of note:

- for IOV Name Service
  - `IMAGE_IOVNS_OPTS` allows you to customize the anti-spam fee, etc.
- for Tendermint
  - `IMAGE_TM_OPTS` allows you to customize the configuration of tendermint, including `priv_validator_laddr`, `p2p.pex`, `p2p.persistent_peers`, `p2p.private_peer_ids`, etc.  **In other words, it's `/etc/systemd/system/iovns.env` that determines whether the node will act as a sentry or validator based on `priv_validator_laddr` and `p2p.*` options.**  Please refer to the <a href="https://tendermint.com/docs/tendermint-core/configuration.html#options" target="blank_">tendermint documentation for the options</a>.

> Old versions of systemd (CentOs) cannot take advantage of the dynamism of `/etc/systemd/system/iovns.env` like modern versions can.  Consequently, the service files themselves are where `bnsd` (IOV Name Service) and `tendermint` options must be specified.

## Point the nodes at each other

Now that you have sentry node(s) and a validator, they need to be made aware of their role and pointed at each other.

### Sentry node configuration

In the most rudimentary form, a sentry node is meant to gossip with other nodes but keep its associated validator hidden.  Change `/etc/systemd/system/iovns.env` so that the node gossips while keeping its validator hidden.  Be mindful of `--rpc.unsafe=true` below, you might not want that.  **On the validator node**, execute `curl -s http://localhost:16657/status | jq -r .result.node_info.id` to get the value for `VALIDATOR_ID`.

```sh
IMAGE_TM_OPTS="\
--moniker='sentry' \
--p2p.seeds=352ba402a2461020689c86cab599c8a44bd49a33@35.198.191.90:26656,88af4a6c555c058e530c124babfa9f9fb12a01b2@35.234.78.200:26656 \
--p2p.pex=true \
--p2p.private_peer_ids='VALIDATOR_ID' \
--rpc.unsafe=true \
"
```

There are a lot more tendermint configuration options available than those shown above.  Customize them as you see fit.

### Validator configuration

As mentioned, it's `/etc/systemd/system/iovns.env` that determines whether the node will act as a sentry or validator based on `p2p.*` options and `priv_validator_laddr` if you're using an HSM.  Change `/etc/systemd/system/iovns.env` so that the node gossips with its sentry node(s) only, ie set **`p2p.pex=false`** and add an explicit list of `p2p.persistent_peers`.  Obtain the sentry node ids for `p2p.persistent_peers` by executing `curl -s http://localhost:16657/status | jq -r .result.node_info.id` **on each sentry node**.  You know the IP and PORT of the nodes, so include them appropriately.

```sh
IMAGE_TM_OPTS="\
--moniker='validator' \
--priv_validator_laddr='tcp://HSM_IP:HSM_PORT' \
--p2p.persistent_peers='SENTRY_ID0@SENTRY_IP0:SENTRY_PORT0,SENTRY_ID1@SENTRY_IP1:SENTRY_PORT1' \
--p2p.pex=false \
--rpc.unsafe=false \
"
```

## Light-up the validator

Once your sentry nodes and validator are sync'ed then the final step to becoming a validator is to submit your validator's pub_key to IOV.  **On your validator node**, execute `curl --silent --fail http://localhost:16657/status | jq -r .result.validator_info.pub_key.value` and reply with the resulting 44 character pub_key to the ticket that was issued to you when you applied for <a href="https://support.iov.one/hc/en-us/requests/new?ticket_form_id=360000417771" target="_blank">the validator program</a>.  (There's no `create-validator` command like in Cosmos; validators are added via governance, which is just IOV on the testnet, for the moment.)
