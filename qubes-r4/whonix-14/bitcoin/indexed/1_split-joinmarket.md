# Qubes 4 & Whonix 14: Split JoinMarket Wallet
Create a two VM system for a fully functional JoinMarket wallet in an offline VM. The `joinmarketd` daemon and `bitcoind` node run in a separate Whonix VM, communicate only over Tor, prefer hidden services, and use stream isolation.
## What is JoinMarket?
Joinmarket is a decentralized, open source, and trustless market for Bitcoin privacy using [coinjoin](https://en.bitcoin.it/wiki/CoinJoin). Anyone holding Bitcoin can offer coinjoins for a fee, and anyone can pay a fee to have their transactions obfuscated.

There is a detailed explanation of the concept by the creator [here](https://bitcointalk.org/index.php?topic=919116.0), and a descriptive infographic [here](https://imgur.com/C6w0Pgf).
## Why Do This?
The benefit of this setup is that your wallet VM is essentially cold storage, yet retains all the functionality of an internet connected JoinMarket wallet (send payment, yield generator, etc).

The only way a remote attacker can compromise this system is to successfully exploit one of your internet connected VMs and then use a Qubes/Xen 0-day to escape that VM.
## Prerequisites
- Have completed [`0_build-bitcoind`](https://github.com/qubenix/guides/blob/master/qubes-r4/whonix-14/bitcoin/indexed/0_build-bitcoind.md) guide.

## I. Set Up Dom0
### A. Create an AppVM.
1. Create the AppVM for JoinMarket's wallet, with no networking and using Bitcoin's TemplateVM.

```
[user@dom0 ~]$ qvm-create --label black --prop netvm='' --template whonix-ws-14-bitcoin joinmarket
```
### B. Enable `joinmarketd` service.
```
[user@dom0 ~]$ qvm-service --enable bitcoind joinmarketd
```
### C. Create rpc policies for comms from `joinmarket` to `bitcoind` VM.
```
[user@dom0 ~]$ echo 'joinmarket bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.{bitcoind,joinmarketd} > /dev/null
```
## II. Set Up TemplateVM
### A. In a `whonix-ws-14-bitcoin` terminal, add `stretch-backports` repo.
```
user@host:~$ sudo su -c "echo -e 'deb tor+http://vwakviie2ienjx6t.onion/debian stretch-backports main' > /etc/apt/sources.list.d/backports.list"
```
### B. Update and install dependencies.
```
user@host:~$ sudo apt-get update && sudo apt-get install -y curl git libsecp256k1-dev \
libsodium-dev python-cffi python-dev python-future python-libnacl python-mnemonic \
python-openssl python-pip python-pycparser python-service-identity python-sip \
python-txtorcon python-virtualenv
```
<!--
NOTE: These packages will be installed via `pip`.
NOTE: Versions checked on: 12-17-2018.
- python-twisted==18.9.0 | buster==18.9.0-3
- python-zope.interface>=4.4.2 | version not in repos
- python-hamcrest>=1.9.0 | version not in repos
- coincurve | not in repos
- argon2_cffi | not in repos
- bencoder.pyx | not in repos
-->
### B. Create system user.
```
user@host:~$ sudo adduser --group --system joinmarketd
Adding system user `joinmarketd' (UID 117) ...
Adding new group `joinmarketd' (GID 123) ...
Adding new user `joinmarketd' (UID 117) with group `joinmarketd' ...
Creating home directory `/home/joinmarketd' ...
```
### C. Use `systemd` to keep `joinmarketd` running.
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
WorkingDirectory=/home/joinmarketd/joinmarket-clientserver
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
user@host:~$ sudo chmod 0644 /lib/systemd/system/joinmarketd.service
```
4. Enable the service.

```
user@host:~$ sudo systemctl enable joinmarketd.service
Created symlink /etc/systemd/system/multi-user.target.wants/joinmarketd.service → /lib/systemd/system/joinmarketd.service.
```
### C. Shutdown TemplateVM.
```
user@host:~$ sudo shutdown now
```
## III. Install JoinMarket
### A. In a `bitcoind` terminal, install JoinMarket and dependencies.
1. Clone [JoinMarket](https://github.com/JoinMarket-Org/joinmarket-clientserver) repository.

```
user@host:~$ git clone https://github.com/JoinMarket-Org/joinmarket-clientserver
Cloning into 'joinmarket-clientserver'...
remote: Enumerating objects: 78, done.
remote: Counting objects: 100% (78/78), done.
remote: Compressing objects: 100% (68/68), done.
remote: Total 3531 (delta 25), reused 43 (delta 10), pack-reused 3453
Receiving objects: 100% (3531/3531), 3.03 MiB | 69.00 KiB/s, done.
Resolving deltas: 100% (2287/2287), done.
```
2. Receive signing key.

```
user@host:~$ gpg --recv-keys 2B6FC204D9BF332D062B461A141001A1AF77F20B
gpg: key 0x141001A1AF77F20B: public key "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify source code.

**Note:** at the time of writing the most recent version of JoinMarket is `v0.4.2`, modify the following steps accordingly if the version has changed.

```
user@host:~$ cd ~/joinmarket-clientserver
user@host:~/joinmarket-clientserver$ git verify-tag v0.4.2
gpg: Signature made Thu 22 Nov 2018 12:51:27 PM UTC
gpg:                using RSA key 141001A1AF77F20B
gpg: Good signature from "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 2B6F C204 D9BF 332D 062B  461A 1410 01A1 AF77 F20B
user@host:~/joinmarket-clientserver$ git checkout -q v0.4.2
```
### B. Get JoinMarket dependencies not available through `apt`.
1. Create `python` virtual environment.

```
user@host:~/joinmarket-clientserver$ virtualenv jmvenv
Running virtualenv with interpreter /usr/bin/python2
New python executable in /home/user/joinmarket-clientserver/jmvenv/bin/python2
Also creating executable in /home/user/joinmarket-clientserver/jmvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
2. Install dependencies to virtual environment.

**Note:** the last command in this section will produce a lot of output. This is normal, be patient.

```
user@host:~/joinmarket-clientserver$ cp -r /usr/lib/python2.7/dist-packages/ jmvenv/lib/python2.7/
user@host:~/joinmarket-clientserver$ source jmvenv/bin/activate
(jmvenv) user@host:~/joinmarket-clientserver$ python setupall.py --all
```
3. Deactivate virtual environment and make relocatable.

```
(jmvenv) user@host:~/joinmarket-clientserver$ deactivate
user@host:~/joinmarket-clientserver$ virtualenv --relocatable jmvenv
```
### C. Relocate `joinmarket-clientserver/` directory.
1. Copy `joinmarket-clientserver/` directory to the `joinmarketd` user's home directory and fix owner.

```
user@host:~/joinmarket-clientserver$ sudo cp -r ~/joinmarket-clientserver/ /home/joinmarketd
user@host:~/joinmarket-clientserver$ sudo chown -R joinmarketd:joinmarketd /home/joinmarketd
```
2. Copy `joinmarket-clientserver/` directory to the `joinmarket` VM.

**Note:** select `joinmarket` from the `dom0` pop-up.

```
user@host:~/joinmarket-clientserver$ qvm-copy ~/joinmarket-clientserver/
```
## IV. Configure `bitcoind` and `joinmarketd`
### A. In a `sys-whonix` terminal, find out the gateway IP.
**Note:** save your gateway IP for later to replace `<gateway-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.xx
```
### B. In a `bitcoind` terminal, create RPC credentials for JoinMarket to communicate with `bitcoind`.
1. Create a random RPC username. Do not use the one shown.

**Note:** save your username for later to replace `<rpc-user>` in examples.

```
user@host:~$ head -c 15 /dev/urandom | base64
uJDzc07zxn5riJDx7N5m
```
2. Use Bitcoin's tool to create a random RPC password and config entry. Do not use the one shown.

**Notes:** 
- Save the hased password (the text on the `rpcauth=` line after `rpcauth=uJDzc07zxn5riJDx7N5m:` in the example) for later to replace `<hashed-pass>` in examples. 
- Save your password for later to replace `<rpc-pass>` in examples. 
- Replace `<rpc-user>` with the information noted earlier.

```
user@host:~$ ~/bitcoin/share/rpcauth/rpcauth.py <rpc-user>
String to be appended to bitcoin.conf:
rpcauth=uJDzc07zxn5riJDx7N5m:838c4dd74606918f1f27a5a2a52b168$9634018b87451bca05082f51b0b5b876fc72ef877bad98298e97e277abd5f90c
Your password:
IuziNnTsOUkonsDD3jn5WatPnFrFOMSnGUsRSUaq5Qg=
```
### C. Configure `bitcoind`.
1. Open `bitcoin.conf`.

```
user@host:~$ sudo -u bitcoind kwrite /home/bitcoind/.bitcoin/bitcoin.conf
```
2. Paste the following at the bottom of the file.

**Note:** be sure not to alter any of the existing information. Replace `<rpc-user>` and `<hashed-pass>` with the information noted earlier.

```
# JoinMarket Auth
rpcauth=<rpc-user>:<hashed-pass>
# JoinMarket Wallet
wallet=joinmarket.dat
```
3. Save the file.

### D. Create `joinmarketd` action file.
```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:127.0.0.1:27183' > /rw/usrlocal/etc/qubes-rpc/qubes.joinmarketd"
```
## VI. Configure `joinmarket` VM
### A. In a `joinmarket` terminal, open communication ports on boot.
1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo kwrite /rw/config/rc.local
```
2. Paste the following at the bottom of the file.

```
socat TCP-LISTEN:8332,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm bitcoind qubes.bitcoind" &
socat TCP-LISTEN:27183,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm bitcoind qubes.joinmarketd" &
```
3. Save the file.
4. Switch back to the `joinmarket` terminal, fix permissions, and execute the file.

```
user@host:~$ sudo chmod 0755 /rw/config/rc.local
user@host:~$ sudo /rw/config/rc.local
```
### B. Source the virtual environment and change directories on boot.
1. Move the directory that was copied over from `bitcoind`.

**Note:** the directory must be moved to your home dir, otherwise the virtual environment will not work.

```
user@host:~$ mv ~/QubesIncoming/bitcoind/joinmarket-clientserver/ ~
```
2. Edit the file `~/.bashrc`.

**Note:** you should not be using the `joinmarket` VM for anything other than your JoinMarket wallet, therefore these changes should be helpful.

```
user@host:~$ kwrite ~/.bashrc & exit
```
3. Paste the following at the bottom of the file.

```
source /home/user/joinmarket-clientserver/jmvenv/bin/activate
cd /home/user/joinmarket-clientserver/scripts/
```
4. Save the file and open a new `joinmarket` terminal.

### C. In a `joinmarket` terminal, configure JoinMarket.
1. Generate a JoinMarket configuration file.

```
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ python wallet-tool.py &>/dev/null
```
2. Make a backup and edit the file `joinmarket.cfg`.

```
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ cp joinmarket.cfg joinmarket.cfg.orig
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ echo > joinmarket.cfg
(jmvenv) user@host:~/joinmarket-clientserver/scripts$ kwrite joinmarket.cfg
```
3. Paste the following, be sure to replace `<gateway-ip>`, `<rpc-user>`, and `<rpc-pass>` with the information noted earlier.

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
port = 6698
usessl = true
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9180
host = 6dvj6v5imhny3anf.onion

[MESSAGING:server2]
# Agora Anarplex IRC
channel = joinmarket-pit
port = 6667
usessl = false
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9181
host = cfyfz6afpgfeirst.onion

[LOGGING]
console_log_level = INFO

[TIMEOUT]
maker_timeout_sec = 60
unconfirm_timeout_sec = 90
confirm_timeout_hours = 6

[POLICY]
segwit = true
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

Once `bitcoind` has finished syncing in the `bitcoind` VM you will be able to use JoinMarket's wallet from the `joinmarket` VM. To learn more about using JoinMarket's wallet please see their [wiki](https://github.com/JoinMarket-Org/joinmarket/wiki).
