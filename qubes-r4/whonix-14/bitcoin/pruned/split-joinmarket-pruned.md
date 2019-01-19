# Qubes 4 & Whonix 14: Split JoinMarket Wallet & Pruned Bitcoin
Create a two VM system for a fully functional JoinMarket wallet in an offline VM. The `joinmarketd` daemon and pruned `bitcoind` node run in a separate Whonix VM, communicate only over Tor, prefer hidden services, and use stream isolation.
## What is JoinMarket?
Joinmarket is a decentralized, open source, and trustless market for Bitcoin privacy using [coinjoin](https://en.bitcoin.it/wiki/CoinJoin). Anyone holding Bitcoin can offer coinjoins for a fee, and anyone can pay a fee to have their transactions obfuscated.

There is a detailed explanation of the concept by the creator [here](https://bitcointalk.org/index.php?topic=919116.0), and a descriptive infographic [here](https://imgur.com/C6w0Pgf).
## Why Do This?
The benefit of this setup is that your wallet VM is essentially cold storage, yet retains all the functionality of an internet connected JoinMarket wallet (send payment, yield generator, etc). The only way a remote attacker can compromise this system is to successfully exploit one of your internet connected VMs and then use a Qubes/Xen 0-day to escape that VM.

In addition to the security improvements, using a pruned Bitcoin node only requires about 10G of disk space.
## I. Set Up Dom0
### A. In a `dom0` terminal, clone Whonix TemplateVM.

```
[user@dom0 ~]$ qvm-clone whonix-ws-14 whonix-ws-14-jm
```
### B. Create a gateway.
**Note:** This gateway should be independent of your normal browsing gateway (`sys-whonix`) to isolate the onion service. You must choose a label color, but it does not have to match this example.

```
[user@dom0 ~]$ qvm-create --label purple --prop netvm='sys-firewall' --prop provides_network='True' --template whonix-gw-14 sys-bitcoind
```
### C. Create two AppVMs, use newly created gateway and template.
1. Create the VM for `bitcoind` and `joinmarketd`, use a Whonix gateway for networking.

**Note:** You must choose a label color, but it does not have to match this example.

```
[user@dom0 ~]$ qvm-create --label red --prop netvm='sys-bitcoind' --template whonix-ws-14-jm jm-bitcoind
```

2. Create the VM for JoinMarket's wallet with no networking.

```
[user@dom0 ~]$ qvm-create --label black --prop netvm='' --template whonix-ws-14-jm jm-wallet
```

3. Resize `jm-bitcoind`.

```
[user@dom0 ~]$ qvm-volume resize jm-bitcoind:private 20G
```
### D. Create rpc policies for comms from `jm-wallet` to `jm-bitcoind`.
```
[user@dom0 ~]$ echo 'jm-wallet jm-bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.{bitcoind,joinmarketd} > /dev/null
```
### E. Enable `bitcoind` and `joinmarketd` services.
```
[user@dom0 ~]$ qvm-service --enable jm-bitcoind bitcoind
[user@dom0 ~]$ qvm-service --enable jm-bitcoind joinmarketd
```
## II. Set Up TemplateVM
### A. In the `whonix-ws-14-jm` terminal, update and install dependencies.
```
user@host:~$ sudo apt update && sudo apt install -y libffi-dev libgmp-dev libsecp256k1-dev libsodium-dev libtool python-virtualenv python3-dev python3-pip
```
<!--
TODO NOTES: Try to limit package installs through pip
-->
### B. Create system users.
1. Add `bitcoind` user.

```
user@host:~$ sudo adduser --group --system bitcoind
Adding system user `bitcoind' (UID 116) ...
Adding new group `bitcoind' (GID 122) ...
Adding new user `bitcoind' (UID 116) with group `bitcoind' ...
Creating home directory `/home/bitcoind' ...
```
2. Add `joinmarketd` user.

```
user@host:~$ sudo adduser --group --system joinmarketd
Adding system user `joinmarketd' (UID 117) ...
Adding new group `joinmarketd' (GID 123) ...
Adding new user `joinmarketd' (UID 117) with group `joinmarketd' ...
Creating home directory `/home/joinmarketd' ...
```
### C. Use `systemd` to keep `bitcoind` always running.
1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /lib/systemd/system/bitcoind.service
```

2. Paste the following.

```
[Unit]
Description=Bitcoin's distributed currency daemon
ConditionPathExists=/var/run/qubes-service/bitcoind
After=qubes-sysinit.service

[Service]
User=bitcoind
Group=bitcoind

Type=forking
PIDFile=/home/user/.bitcoin/bitcoind.pid
ExecStart=/usr/local/bin/bitcoind
ExecStop=/usr/local/bin/bitcoin-cli stop

Restart=always
PrivateTmp=true
TimeoutStopSec=60s
TimeoutStartSec=2s
StartLimitInterval=120s
StartLimitBurst=5

[Install]
WantedBy=multi-user.target
```

3. Save the file and switch back to the terminal.

### D. Use `systemd` to keep `joinmarketd` always running.
1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /lib/systemd/system/joinmarketd.service
```
2. Paste the following.

```
[Unit]
Description=JoinMarket's server daemon
ConditionPathExists=/var/run/qubes-service/joinmarketd
After=qubes-sysinit.service

[Service]
User=joinmarketd
Group=joinmarketd

Type=idle
WorkingDirectory=/home/joinmarketd/joinmarket-clientserver-0.5.2
ExecStart=/bin/sh -c 'jmvenv/bin/python scripts/joinmarketd.py > scripts/logs/joinmarketd.log'

Restart=always
PrivateTmp=true
TimeoutStopSec=60s
TimeoutStartSec=2s
StartLimitInterval=120s
StartLimitBurst=5

[Install]
WantedBy=multi-user.target
```
3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ sudo chmod 0644 /lib/systemd/system/{bitcoind,joinmarketd}.service
```
4. Enable the services.

```
user@host:~$ sudo systemctl enable {bitcoind,joinmarketd}.service
Created symlink /etc/systemd/system/multi-user.target.wants/bitcoind.service → /lib/systemd/system/bitcoind.service.
Created symlink /etc/systemd/system/multi-user.target.wants/joinmarketd.service → /lib/systemd/system/joinmarketd.service.
```
### E. Shutdown TemplateVM.
```
user@host:~$ sudo shutdown now
```
## II. Install Bitcoin and JoinMarket
### A. In a `jm-bitcoind` terminal, install Bitcoin.
1. Download and verify the Linux 64 bit version of [Bitcoin Core](https://bitcoincore.org/en/download/).

**Note:** at the time of writing the most recent version of Bitcoin Core is `0.17.1`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -O "https://bitcoincore.org/bin/bitcoin-core-0.17.1/bitcoin-0.17.1-x86_64-linux-gnu.tar.gz" -O "https://bitcoincore.org/bin/bitcoin-core-0.17.1/SHA256SUMS.asc"
user@host:~$ gpg --recv-keys 01EA5486DE18A882D4C2684590C8019E36C2E964
gpg: keybox '/home/user/.gnupg/pubring.kbx' created
key 0x90C8019E36C2E964:
57 signatures not checked due to missing keys
gpg: /home/user/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x90C8019E36C2E964: public key "Wladimir J. van der Laan (Bitcoin Core binary release signing key) <laanwj@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
user@host:~$ gpg --verify SHA256SUMS.asc
gpg: Signature made Tue 25 Dec 2018 08:03:05 AM UTC
gpg:                using RSA key 0x90C8019E36C2E964
gpg: Good signature from "Wladimir J. van der Laan (Bitcoin Core binary release signing key) <laanwj@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 01EA 5486 DE18 A882 D4C2  6845 90C8 019E 36C2 E964
user@host:~$ grep x86 SHA256SUMS.asc | shasum -c
bitcoin-0.17.1-x86_64-linux-gnu.tar.gz: OK
```
2. Extract and install.

```
user@host:~$ tar xf bitcoin-0.17.1-x86_64-linux-gnu.tar.gz
user@host:~$ sudo install -g staff -m 0755 -o root -t /usr/local/bin/ bitcoin-0.17.1/bin/bitcoin*
```
### B. Install JoinMarket.
1. Download and verify [JoinMarket](https://github.com/JoinMarket-Org/joinmarket-clientserver/releases).

**Note:** at the time of writing the most recent version of JoinMarket is `v0.5.2`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -LO "https://github.com/JoinMarket-Org/joinmarket-clientserver/archive/v0.5.2.tar.gz" -O "https://github.com/JoinMarket-Org/joinmarket-clientserver/releases/download/v0.5.2/joinmarket-clientserver-0.5.2.tar.gz.asc"
user@host:~$ gpg --verify joinmarket-clientserver-0.5.2.tar.gz.asc v0.5.2.tar.gz
gpg: Signature made Sat 19 Jan 2019 07:48:07 PM UTC
gpg:                using RSA key 0x141001A1AF77F20B
gpg: Good signature from "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 2B6F C204 D9BF 332D 062B  461A 1410 01A1 AF77 F20B
```
2. Extract and enter JoinMarket directory.

```
user@host:~$ tar xf v0.5.2.tar.gz
user@host:~$ cd joinmarket-clientserver-0.5.2
```
3. Create python virtual environment.

```
user@host:~/joinmarket-clientserver-0.5.2$ virtualenv -p python3 jmvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/user/joinmarket-clientserver-0.5.2/jmvenv/bin/python3
Also creating executable in /home/user/joinmarket-clientserver-0.5.2/jmvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
4. Install dependencies to virtual environment.

**Note:** this will produce a lot of output. This is normal, be patient.

```
user@host:~/joinmarket-clientserver-0.5.2$ source jmvenv/bin/activate
(jmvenv) user@host:~/joinmarket-clientserver-0.5.2$ python setupall.py --all
```
5. Deactivate virtual environment and make relocatable.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.2$ deactivate
user@host:~/joinmarket-clientserver-0.5.2$ virtualenv -p python3 --relocatable jmvenv
```
### B. Copy `joinmarket-clientserver/` directory.
1. Copy `joinmarket-clientserver-0.5.2/` directory to the `joinmarketd` user's home directory and fix owner.

```
user@host:~/joinmarket-clientserver-0.5.2$ sudo cp -r ~/joinmarket-clientserver-0.5.2/ /home/joinmarketd
user@host:~/joinmarket-clientserver-0.5.2$ sudo chown -R joinmarketd:joinmarketd /home/joinmarketd
```
2. Copy `joinmarket-clientserver-0.5.2/` directory to the `jm-wallet` VM.

**Note:** select `jm-wallet` from the `dom0` pop-up.

```
user@host:~/joinmarket-clientserver-0.5.2$ qvm-copy ~/joinmarket-clientserver-0.5.2/
```
## III. Configure Gateway
### A. In a `sys-bitcoind` terminal, find out the gateway IP.
**Note:** save your gateway IP for later to replace `<gateway-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.xx
```
### B. Configure `onion-grater`.
1. Install provided profile for `bitcoind` to persistent directory.

```
user@host:~$ sudo install -D -t /usr/local/etc/onion-grater-merger.d/ /usr/share/onion-grater-merger/examples/40_bitcoind.yml
```
2. Restart `onion-grater` service.

```
user@host:~$ sudo systemctl restart onion-grater.service
```
## IV. Configure `bitcoind` and `joinmarketd`
### A. In a `jm-bitcoind` terminal, create RPC credentials.
1. Create a random RPC username. Do not use the one shown.

**Note:** save your username for later to replace `<rpc-user>` in examples.

```
user@host:~$ head -c 15 /dev/urandom | base64
uJDzc07zxn5riJDx7N5m
```
2. Create a random RPC password. Do not use the one shown.

**Note:** save your password for later to replace `<rpc-pass>` in examples.

```
user@host:~$ head -c 30 /dev/urandom | base64
pGw0+tSSzxwCCkJIjDHFg8Jezn1d7Yc7WkksPlQ0
```
### C. Configure `bitcoind`.
1. Create Bitcoin's data directory and configuration file.

```
user@host:~$ mkdir -m 0700 ~/.bitcoin
user@host:~$ kwrite ~/.bitcoin/bitcoin.conf
```
2. Paste the following, be sure to replace `<gateway-ip>`, `<rpc-user>`, and `<rpc-pass>` with the information noted earlier.

**Note:** do not discard your note of these credentials, you will need them again.

```
daemon=1
listen=1
onion=<gateway-ip>:9111
onlynet=onion
proxy=<gateway-ip>:9111
prune=550
rpcuser=<rpc-user>
rpcpassword=<rpc-pass>
wallet=jm-wallet.dat
```
3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ chmod 0600 ~/.bitcoin/bitcoin.conf
```
## V. Create Communication Channels
### A. In a `jm-bitcoind` terminal, create `qubes-rpc` action files.
1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```
2. Create `bitcoind` action file.

```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:127.0.0.1:8332' > /rw/usrlocal/etc/qubes-rpc/qubes.bitcoind"
```
3. Create `joinmarketd` action file.

```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:127.0.0.1:27183' > /rw/usrlocal/etc/qubes-rpc/qubes.joinmarketd"
```
## VI. Configure JoinMarket.
### A. In a `jm-wallet` terminal, open communication ports on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo kwrite /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
socat TCP-LISTEN:8332,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm jm-bitcoind qubes.bitcoind" &
socat TCP-LISTEN:27183,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm jm-bitcoind qubes.joinmarketd" &
```
3. Save the file. Then switch back to the terminal, fix permissions, and execute the file.

```
user@host:~$ sudo chmod 0755 /rw/config/rc.local
user@host:~$ sudo /rw/config/rc.local
```
### B. Source the virtual environment and change directories on boot.
1. Move the directory that was copied over from `jm-bitcoind`.

```
user@host:~$ mv ~/QubesIncoming/jm-bitcoind/joinmarket-clientserver-0.5.2/ ~
```
2. Edit the file `~/.bashrc`.

**Note:** you should not be using the `jm-wallet` VM for anything other than your JoinMarket wallet, therefore these changes should be helpful.

```
user@host:~$ kwrite ~/.bashrc & exit
```
3. Paste the following at the bottom of the file.

```
source /home/user/joinmarket-clientserver-0.5.2/jmvenv/bin/activate
cd /home/user/joinmarket-clientserver-0.5.2/scripts/
```
4. Save the file and open a new `jm-wallet` terminal.

### C. In a `jm-wallet` terminal, configure JoinMarket.
1. Generate a JoinMarket configuration file.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.2/scripts$ python wallet-tool.py
Created a new `joinmarket.cfg`. Please review and adopt the settings and restart joinmarket.
```
2. Make a backup and edit the file `joinmarket.cfg`.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.2/scripts$ cp joinmarket.cfg joinmarket.cfg.orig
(jmvenv) user@host:~/joinmarket-clientserver-0.5.2/scripts$ echo > joinmarket.cfg
(jmvenv) user@host:~/joinmarket-clientserver-0.5.2/scripts$ kwrite joinmarket.cfg
```
3. Paste the following.

**Note:** be sure to replace all instances of `<gateway-ip>`, `<rpc-user>`, and `<rpc-pass>` with the information noted earlier.

```
[DAEMON]
no_daemon = 0
daemon_port = 27183
daemon_host = 127.0.0.1
use_ssl = false

[BLOCKCHAIN]
blockchain_source = bitcoin-rpc
network = mainnet
rpc_host = 127.0.0.1
rpc_port = 8332
rpc_user = <rpc-user>
rpc_password = <rpc-pass>
rpc_wallet_file = joinmarket.dat

[MESSAGING:server1]
# Cyberguerrilla IRC
channel = joinmarket-pit
host = 6dvj6v5imhny3anf.onion
port = 6698
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9180
usessl = true

[MESSAGING:server2]
# Agora Anarplex IRC
channel = joinmarket-pit
host = cfyfz6afpgfeirst.onion
port = 6667
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9181
usessl = false

[MESSAGING:server3]
# DarkScience IRC
channel = joinmarket-pit
host = darksci3bfoka7tw.onion
port = 6697
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9182
usessl = true

[LOGGING]
console_log_level = INFO
color = true

[TIMEOUT]
maker_timeout_sec = 60
unconfirm_timeout_sec = 180
confirm_timeout_hours = 6

[POLICY]
segwit = true
native = false
# merge_algorithm = greediest, greedy, gradual, default
merge_algorithm = default
# tx_fees = less than 144 is block time, more than 144 is satoshi/Kb
tx_fees = 3
absurd_fee_per_kb = 350000
tx_broadcast = self
minimum_makers = 2
taker_utxo_retries = 3
taker_utxo_age = 5
taker_utxo_amtpercent = 20
accept_commitment_broadcasts = 1
```
4. Save the file.

## VI. Finish
The guide is complete.

Once `bitcoind` has finished syncing in the `jm-bitcoind` VM you will be able to use JoinMarket's wallet from the `jm-wallet` VM. To learn more about using JoinMarket's wallet please see their [wiki](https://github.com/JoinMarket-Org/joinmarket/wiki).
