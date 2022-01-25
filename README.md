ecurve on EOS at block 219608363: data snapshot
===============================================

The goal of this task is to provide a complete snapshot of eCurve
balances at the block number 219608363 (timestamp:
2021-12-08T12:35:53.500 UTC), as the finances were drained because of
a security attack soon afterwards.

The method is as follows:

1. A dedicated server or a VPS is set up, with EOS mainned node and
state history that includes the snapshot block number. The EOS node
will let run for a short period of time in order to have the required
information.

2. State history output is processed by Chronicle software. The state
history archive contains all contract tables content, and Chronicle
decodes this information in JSON format. 

3. A consumer script reads the Chronicle output and stores the
relevenat data in MySQL database. It stops at the specified block.

4. A complete dump of the MySQL database becomes available for
analyzis and balance verification.


Software used in this processing:

1. Chronicle, commit 41b0150d19399e4b5ed4e01f82ea371a246fc326
https://github.com/EOSChronicleProject/eos-chronicle

2. Chronicle consumer for table contents, commit
2a97ab4d96b8e5ed94acede074df93e52054bfe8
https://github.com/cc32d9/eosio_tables_db


Server setup and data extraction
--------------------------------

A fresh VPS is provisioned with 32GB RAM and Ubuntu 20.04. At least
50GB storage is required. The build instructions below are assuming 8
CPU cores.

```
apt update && apt install -y git zstd curl

# protect TCP ports 8000-8999 from outside
iptables -A INPUT -i ens2 -p tcp -m tcp --dport 8000:8999 -j DROP

# install eosio 2.0.13. The binary package is built for Ubuntu 18.04,
# and requires libicu from it. The required package can be manually
# downloaded and installed.

cd /var/local

wget wget http://archive.ubuntu.com/ubuntu/pool/main/i/icu/libicu60_60.2-3ubuntu3_amd64.deb 
apt install -y ./libicu60_60.2-3ubuntu3_amd64.deb 

wget wget https://github.com/EOSIO/eos/releases/download/v2.0.13/eosio_2.0.13-1-ubuntu-18.04_amd64.deb
apt install -y ./eosio_2.0.13-1-ubuntu-18.04_amd64.deb


## EOS node configuration with a single p2p peer

mkdir -p /srv/eos/etc
cat >/srv/eos/etc/config.ini <<'EOT'
chain-state-db-size-mb = 65536
reversible-blocks-db-size-mb = 2048
wasm-runtime = eos-vm-jit
eos-vm-oc-enable = true
http-server-address = 0.0.0.0:8888
p2p-listen-endpoint = 0.0.0.0:9856
access-control-allow-origin = *
validation-mode = light
read-mode = head
plugin = eosio::chain_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::state_history_plugin
trace-history = true
chain-state-history = true
state-history-endpoint = 0.0.0.0:8080
p2p-peer-address = mainnet.eosamsterdam.net:9876
EOT



## systemd unit configuration file for starting and stopping the node

cat >/etc/systemd/system/eos.service <<'EOT'
[Unit]
Description=EOS mainnet node
[Service]
Type=simple
ExecStart=/usr/bin/nodeos --data-dir /srv/eos/data --config-dir /srv/eos/etc --disable-replay-opts
TimeoutStartSec=30s
TimeoutStopSec=300s
Restart=no
User=root
Group=daemon
KillMode=control-group
[Install]
WantedBy=multi-user.target
EOT

systemctl daemon-reload


# retrieve an EOS snapshot that is earlier than the target block
# Snapshot sources:
#   https://snapshots.eosnation.io/
#   http://snapshots.eossweden.org/

cd /var/local
curl https://snapshots-cdn.eossweden.org/main/2.0/snapshot-219519257.bin.tar.gz | tar xzvf -

# Start the node from the snapshot, monitor it with "cleos get info",
# and stop the node with Ctrl-C as soon as last irreversible block is
# higher than 219608363. The startup takes about 10-15 minutes befor
# ethe node starts responding on its HTTP interface.


/usr/bin/nodeos --data-dir /srv/eos/data --config-dir /srv/eos/etc --disable-replay-opts --snapshot snapshot-219519257.bin


# after stopping the node, remove p2p peers from its configuration

sed -i -e 's,^p2p-peer-address,\#p2p-peer-address,g' /srv/eos/etc/config.ini

# start the node as a service. It will not move any further in
# blockchain history and not consume much resource on the VPS.

systemctl start eos

# note the first block number in state history, as it will be required
# to start the processing.

journalctl -u eos | grep chain_state_history.log
# output: chain_state_history.log has blocks 219519258-219612896


####  install Chronicle software ###

apt update && apt install -y gnupg software-properties-common curl
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | apt-key add -
apt-add-repository 'deb https://apt.kitware.com/ubuntu/ focal main'
add-apt-repository -y ppa:ubuntu-toolchain-r/test

apt update && apt install -y git g++-8 cmake libssl-dev libgmp-dev zlib1g-dev
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 800 --slave /usr/bin/g++ g++ /usr/bin/g++-8

mkdir /opt/src && cd /opt/src && \
wget -O - https://boostorg.jfrog.io/artifactory/main/release/1.67.0/source/boost_1_67_0.tar.gz | tar xzvf -  && \
cd boost_1_67_0 && ./bootstrap.sh && ./b2 install -j 8

cd /opt/src
git clone https://github.com/EOSChronicleProject/eos-chronicle.git
cd eos-chronicle && git submodule update --init --recursive && mkdir build && cd build && cmake .. && make -j 8 install

cp /opt/src/eos-chronicle/systemd/chronicle_receiver\@.service /etc/systemd/system/
systemctl daemon-reload


# chronicle configuration that is only exporting tables for the needed contract

mkdir -p /srv/eosio_tables_eos/chronicle-config/
cat >/srv/eosio_tables_eos/chronicle-config/config.ini <<'EOT'
host = 127.0.0.1
port = 8080
mode = scan
receiver-state-db-size = 16384
skip-block-events = true
irreversible-only = true
plugin = exp_ws_plugin
exp-ws-host = 127.0.0.1
exp-ws-port = 8001
exp-ws-bin-header = true
skip-traces = true
skip-account-info = true
enable-tables-filter = true
include-tables-contract = ecurvelp1111
include-tables-contract = depositlp111
include-tables-contract = depositlp112
include-tables-contract = depositlp113
include-tables-contract = depositlp114
include-tables-contract = dadusdtokens
include-tables-contract = tethertether
include-tables-contract = swap.defi
include-tables-contract = lptoken.defi
include-tables-contract = dpositboxauq
include-tables-contract = dpositboxavg
include-tables-contract = dpositboxayo
include-tables-contract = dpositboxayp
EOT



### set up the database writer ###

apt install -y cpanminus libjson-xs-perl libjson-perl libmysqlclient-dev libdbi-perl \
 libwww-perl make gcc libredis-fast-perl  mariadb-server mariadb-client

cpanm --notest Net::WebSocket::Server
cpanm --notest DBD::MariaDB

git clone https://github.com/cc32d9/eosio_tables_db.git /opt/eosio_tables_db


cd /opt/eosio_tables_db/
mysql <eosio_tables_dbsetup.sql
mysql eosio_tables <eosio_tables_dbtables.sql


for c in \
ecurvelp1111 \
depositlp111 \
depositlp112 \
depositlp113 \
depositlp114 \
dadusdtokens \
tethertether \
swap.defi \
lptoken.defi \
dpositboxauq \
dpositboxavg \
dpositboxayo \
dpositboxayp \
; do mysql eosio_tables --execute="INSERT INTO WATCH_CONTRACTS (network, contract) VALUES ('eos', '"$c"')"; done



# the consumer will stop automatically at the target block
nohup /usr/bin/perl /opt/eosio_tables_db/eosio_tables_dbwriter.pl --network=eos --port=8001 --ack=1 --endblock=219608363 &

# chronicle will stop as soon as consumer stops. The start block must
# match the first block in state history
/usr/local/sbin/chronicle-receiver --config-dir=/srv/eosio_tables_eos/chronicle-config \
--data-dir=/srv/eosio_tables_eos/chronicle-data --start-block=219519258

mkdir /root/report
cd /root/report

# save the data for archiving, a copy is in report_data folder
mysqldump --databases eosio_tables >ecurve_snapshot_219608363.sqldump
gzip ecurve_snapshot_219608363.sqldump

# balance reports, in tab-separated value files


## ecurvelp1111 ##

# all balances in ecurvelp1111, like "alxbransby12    0.000000976 EOSDTPL"
mysql eosio_tables --batch --execute="select scope as account, SUBSTRING_INDEX(fval, ' ', 1) as balance, SUBSTRING_INDEX(fval, ' ', -1) as currency from eos_ROWS where contract='ecurvelp1111' and tbl='accounts' order by scope" >ecurvelp1111_accounts.tsv


## depositlp11* ##

# deposits, like "romans1x2abc    depositlp113    30.315458006 EOSDTPL"
mysql eosio_tables --batch --execute="select account, contract, SUBSTRING_INDEX(fval, ' ', 1) as balance, SUBSTRING_INDEX(fval, ' ', -1) as currency from eos_ROWS, (select pk, contract as ctr, fval as account from eos_ROWS where contract like 'depositlp11%' and tbl='stake' and field='account') X where X.pk=eos_ROWS.pk  and ctr=contract and tbl='stake' and field='balance' order by account,contract" >deposits.tsv


## dadusdtokens ##

# all balances in dadusdtokens, like "antidankguns    6.041695 USDC"
mysql eosio_tables --batch --execute="select scope as account, SUBSTRING_INDEX(fval, ' ', 1) as balance, SUBSTRING_INDEX(fval, ' ', -1) as currency from eos_ROWS where contract='dadusdtokens' and tbl='accounts' order by scope" >dadusdtokens_accounts.tsv


## tethertether ##

# all USDT balances, numbers only 
mysql eosio_tables --batch --execute="select scope as account, SUBSTRING_INDEX(fval, ' ', 1) as balance, SUBSTRING_INDEX(fval, ' ', -1) as currency from eos_ROWS where contract='tethertether' and tbl='accounts' order by scope" >tethertether_balances.tsv


## swap.defi ##

# swap.defi pairs
mysql eosio_tables --batch --execute="select pk,field,fval from eos_ROWS where contract='swap.defi' and tbl='pairs' and pk IN (1255, 1239, 1341, 1342) order by pk, field" > swap.defi_pairs.tsv


## lptoken.defi - TODO ##

cat >symbol_name.sql <<'EOT'
CREATE TABLE SYMBOL_NAME_CONV
( sym VARCHAR(7) PRIMARY KEY,
  name VARCHAR(13) NOT NULL ) ENGINE=InnoDB;
INSERT INTO SYMBOL_NAME_CONV (sym, name) VALUES
('BOXAVG', '...4ipm1f1bo2'),
('BOXAUQ', '...52pe1f1bo2'),
('BOXAYO', '...4yqe1f1bo2'),
('BOXAYP', '...5.qe1f1bo2');
EOT

mysql eosio_tables <symbol_name.sql

mysql eosio_tables --batch --execute="select SYMBOL_NAME_CONV.sym as currency, account, fval as balance from eos_ROWS, (select pk, contract as ctr, scope as scp, fval as account from eos_ROWS where contract='lptoken.defi' and tbl='userinfo' and field='owner') X, SYMBOL_NAME_CONV where X.pk=eos_ROWS.pk and ctr=contract and scope=scp and tbl='userinfo' and field='liquidity' and scope=SYMBOL_NAME_CONV.name order by currency,account" >lptoken.defi_liquidity.tsv


## dpositboxa** ##

# balances of BOXAUQ, BOXAVG, ...
for c in dpositboxauq dpositboxavg dpositboxayo dpositboxayp; do
mysql eosio_tables --batch --execute="select account, SUBSTRING_INDEX(fval, ' ', 1) as balance, SUBSTRING_INDEX(fval, ' ', -1) as currency from eos_ROWS, (select contract as ctr, pk, fval as account from eos_ROWS where contract='"${c}"' and scope='"${c}"' and tbl='stake' and field='account') X where X.ctr=contract and X.pk=eos_ROWS.pk  and scope='"${c}"' and tbl='stake' and field='balance' order by account" >${c}_balances.tsv;
done

```

All `*.tsv` files are available in `report_data/tsv_data.zip` in this repository.

They are also imported into `ecurve_snapshot_219608363.ods` using LibreOffice Calc.

----
2022-01-25 cc32d9@gmail.com





