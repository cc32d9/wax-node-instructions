# WAX state history node at AWS

```
Canonical, Ubuntu, 20.04 LTS, amd64 focal image build on 2022-06-10
Instance type: r5b.4xlarge (128GB RAM, Intel Xeon Platinum 8259CL)
/dev/nvme1n1: 200 GiB swap
/dev/nvme2n1: 1000 GiB data

sudo -i
timedatectl set-timezone UTC
hostnamectl set-hostname aws-wax-ship-01.binfra.one
localectl set-locale LANG=C.UTF-8

apt-get update && apt-get install -y git zfsutils-linux sysstat ntp net-tools curl zstd

mkswap /dev/nvme1n1
cat >> /etc/fstab <<'EOT' 
/dev/nvme1n1       none            swap            defaults
EOT

zpool create -f -o ashift=12 zdata /dev/nvme2n1
zfs set atime=off zdata 
zfs set compression=lz4 zdata 

# reboot to make sure swap and ZFS are initialized properly at boot time
reboot

# log in again via ssh
sudo -i

# verify that swap and ZFS are in place
free -g
zpool list

# prepare the WAX API storage location
zfs create -o mountpoint=/srv/waxship01 zdata/waxship01

# state history data is already compressed by the node, so no need for ZFS compression
zfs create -o mountpoint=/srv/waxship01/data/state-history -o compression=off zdata/waxship01-state-history

# blockchain state in tmpfs
mkdir -p /srv/waxship01/data/state
cat >>/etc/fstab <<'EOT'
tmpfs   /srv/waxship01/data/state  tmpfs rw,nodev,nosuid,size=150G,x-systemd.after=zfs-mount.service 0  0
EOT
mount -a

# install the WAX nodeos package
cd /zdata
wget https://apt.eossweden.org/wax/pool/testing/w/wax/wax_2.0.13wax03-1-ubuntu-18.04_amd64.deb
wget http://security.ubuntu.com/ubuntu/pool/main/i/icu/libicu60_60.2-3ubuntu3.2_amd64.deb
apt install -y ./libicu60_60.2-3ubuntu3.2_amd64.deb ./wax_2.0.13wax03-1-ubuntu-18.04_amd64.deb

# systemd service 
cat >/etc/systemd/system/ship\@.service <<'EOT'
[Unit]
Description=nodeos
[Service]
Type=simple
ExecStart=/usr/bin/nodeos --data-dir /srv/%i/data --config-dir /srv/%i/etc --disable-replay-opts
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

## WAX node configuration. Pick the nearest p2p peers from https://validate.eosnation.io/wax/reports/config.html

mkdir -p /srv/waxship01/etc/
cat >/srv/waxship01/etc/config.ini<<'EOT'
chain-state-db-size-mb = 150000
reversible-blocks-db-size-mb = 2048
read-mode = head
wasm-runtime = eos-vm-jit
eos-vm-oc-enable = true
validation-mode = light
max-transaction-time = 100
http-max-response-time-ms = 100
abi-serializer-max-time-ms = 12000
http-server-address = 0.0.0.0:8901
p2p-listen-endpoint = 0.0.0.0:8911
http-validate-host = false
access-control-allow-origin = *
access-control-allow-headers = Origin, X-Requested-With, Content-Type, Accept
plugin = eosio::chain_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::state_history_plugin
trace-history = true
chain-state-history = true
state-history-endpoint = 0.0.0.0:8921
sync-fetch-span = 100
p2p-peer-address = wax.eu.eosamsterdam.net:9101
p2p-peer-address = peer1.wax.blacklusion.io:4646
p2p-peer-address = peer1-emea.wax.blacklusion.io:4646
EOT

cat > /srv/waxship01/etc/logging.json << 'EOT'
{
  "includes": [],
  "appenders": [{
      "name": "consoleout",
      "type": "console",
      "args": {
        "stream": "std_out",
        "level_colors": [{
            "level": "debug",
            "color": "green"
          },{
            "level": "warn",
            "color": "brown"
          },{
            "level": "error",
            "color": "red"
          }
        ]
      },
      "enabled": true
    }
  ],
  "loggers": [{
      "name": "default",
      "level": "warn",
      "enabled": true,
      "additivity": false,
      "appenders": [
        "consoleout"
      ]
    }
  ]
}
EOT

# download and unpack the latest WAX snapshot from EOS Nation
# alternative snapshot location: https://snapshots.eossweden.org/

cd /zdata
curl -L https://snapshots.eosnation.io/wax-v4/latest | zstd -d >wax_latest_snapshot.bin

# Initialize the node in the background. The initialization may take about an hour.
# Swap memory usage is over 88GB in peak
nohup /usr/bin/nodeos --data-dir /srv/waxship01/data --config-dir /srv/waxship01/etc --disable-replay-opts --snapshot=/zdata/wax_latest_snapshot.bin &

# check if the node is up and starts lustening on 8901
netstat -an --tcp

# after it opens the TCP ports, it starts writing the first state history block. This requires a lot of CPU and RAM.
# State history files are empty for the first 15-20 minutes:
ls -al /srv/waxship01/data/state-history/

Once it started responding on get_info queries, it can be stopped.
cleos -u http://127.0.0.1:8901 get info


# stop the nodeos in the background. Give it some time to free up 88GB from swap
kill %1

# start the node as a service. It will be slow in the beginning, as most of the state is 
# swapped out and needs time to get back into RAM
systemctl start ship@waxship01

# check the resync performance (roughly, 15 blocks per second while 70GB is still in swap)
cleos -u http://127.0.0.1:8901 get info; sleep 60; cleos -u http://127.0.0.1:8901 get info

```
