# WAX API node at AWS

```
Canonical, Ubuntu, 20.04 LTS, amd64 focal image build on 2022-06-10
Instance type: z1d.3xlarge  (approx. $1000/month)
CPU: Intel(R) Xeon(R) Platinum 8151 CPU @ 3.40GHz
Root volume: /dev/nvme0n1: 8 GiB
Data volume: /dev/nvme1n1: 419.1 GiB

sudo -i
timedatectl set-timezone UTC
hostnamectl set-hostname aws-wax-api-01.binfra.one
localectl set-locale LANG=C.UTF-8

apt-get update && apt-get install -y git zfsutils-linux sysstat ntp net-tools curl zstd

# this creates 96GB swap partition and the rest goes to ZFS
parted /dev/nvme1n1 -s -- mklabel gpt mkpart P1 linux-swap 2048s 96Gib mkpart P2 zfs 96Gib -64s

mkswap /dev/nvme1n1p1
cat >> /etc/fstab <<'EOT' 
/dev/nvme1n1p1       none            swap            defaults
EOT

zpool create -f -o ashift=12 zdata /dev/nvme1n1p2
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
zfs create -o mountpoint=/srv/waxapi01 zdata/waxapi01

# blockchain state in tmpfs
mkdir -p /srv/waxapi01/data/state
cat >>/etc/fstab <<'EOT'
tmpfs   /srv/waxapi01/data/state  tmpfs rw,nodev,nosuid,size=150G,x-systemd.after=zfs-mount.service 0  0
EOT
mount -a

# install the WAX nodeos package
cd /zdata
wget https://apt.eossweden.org/wax/pool/testing/w/wax/wax_2.0.13wax03-1-ubuntu-18.04_amd64.deb
wget http://security.ubuntu.com/ubuntu/pool/main/i/icu/libicu60_60.2-3ubuntu3.2_amd64.deb
apt install -y ./libicu60_60.2-3ubuntu3.2_amd64.deb ./wax_2.0.13wax03-1-ubuntu-18.04_amd64.deb

# systemd service 
cat >/etc/systemd/system/nodeos\@.service <<'EOT'
[Unit]
Description=nodeos
[Service]
Type=simple
ExecStart=/usr/bin/nodeos --data-dir /srv/%i/data --config-dir /srv/%i/etc
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

mkdir -p /srv/waxapi01/etc/
cat >/srv/waxapi01/etc/config.ini<<'EOT'
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
sync-fetch-span = 100
p2p-peer-address = wax.eu.eosamsterdam.net:9101
p2p-peer-address = peer1.wax.blacklusion.io:4646
p2p-peer-address = peer1-emea.wax.blacklusion.io:4646
EOT

cat > /srv/waxapi01/etc/logging.json << 'EOT'
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

# initialize the node in the background. As soon as it starts answering on its HTTP port, stop it with kill command
# the initialization may take a half an hour
nohup /usr/bin/nodeos --data-dir /srv/waxapi01/data --config-dir /srv/waxapi01/etc --snapshot=/zdata/wax_latest_snapshot.bin &

# check if the node is up and starts lustening on 8901
netstat -an --tcp
cleos -u http://127.0.0.1:8901 get info

# stop the nodeos in the background
kill %1

# start the node as a service
systemctl start nodeos@waxapi01

# check the resync performance (roughly, 30 blocks per second)
cleos -u http://127.0.0.1:8901 get info; sleep 60; cleos -u http://127.0.0.1:8901 get info
```
