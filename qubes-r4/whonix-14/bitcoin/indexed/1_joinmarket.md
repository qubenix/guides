# Qubes 4 & Whonix 14: Split JoinMarket Wallet
Create a two VM system for a fully functional JoinMarket wallet in an offline VM. The `joinmarketd` daemon and `bitcoind` node run in a separate Whonix VM, communicate only over Tor, prefer hidden services, and use stream isolation.
## What is JoinMarket?
Joinmarket is a decentralized, open source, and trustless market for Bitcoin privacy using [coinjoin](https://en.bitcoin.it/wiki/CoinJoin). Anyone holding Bitcoin can offer coinjoins for a fee, and anyone can pay a fee to have their transactions obfuscated.

There is a detailed explanation of the concept by the creator [here](https://bitcointalk.org/index.php?topic=919116.0), and a descriptive infographic [here](https://imgur.com/C6w0Pgf).
## Why Do This?
This method keeps the wallet VM offline, yet retains all the functionality of an internet connected JoinMarket wallet (send payments, run yield generator/tumbler, etc.).

The only way a remote attacker can compromise this system is to successfully exploit one of your internet connected VMs and then use a Qubes/Xen 0-day to escape that VM.
## Prerequisites
- Have completed [`0_bitcoind`](https://github.com/qubenix/guides/blob/master/qubes-r4/whonix-14/bitcoin/indexed/0_bitcoind.md) guide.

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
### C. Create rpc policies to allow comms from `joinmarket` to `bitcoind` VM.
```
[user@dom0 ~]$ echo 'joinmarket bitcoind allow' | sudo tee -a /etc/qubes-rpc/policy/qubes.{bitcoind,joinmarketd-2718{3,4}} > /dev/null
```
## II. Set Up TemplateVM
### A. Update and install dependencies.
```
user@host:~$ sudo apt update && sudo apt install -y libffi-dev libgmp-dev libsecp256k1-dev libsodium-dev \
python-virtualenv python3-dev python3-pip
```
<!--
TODO NOTES: Try to limit package installs through pip
-->
### B. Create system user.
```
user@host:~$ sudo adduser --system joinmarket
Adding system user `joinmarket' (UID 117) ...
Adding new user `joinmarket' (UID 117) with group `nogroup' ...
Creating home directory `/home/joinmarket' ...
```
### C. Use `systemd` to keep `joinmarketd` running.
1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /lib/systemd/system/joinmarketd.service
```
2. Paste the following.

```
[Unit]
Description=JoinMarket daemon
ConditionPathExists=/var/run/qubes-service/joinmarketd
After=qubes-sysinit.service
After=bitcoind.service

[Service]
WorkingDirectory=/home/joinmarket/joinmarket-clientserver-0.5.3
ExecStart=/bin/sh -c 'jmvenv/bin/python scripts/joinmarketd.py'

RuntimeDirectory=joinmarketd
User=joinmarket
Type=idle
PIDFile=/run/joinmarketd/joinmarketd.pid
Restart=on-failure

PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
MemoryDenyWriteExecute=true

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
### A. In a `bitcoind` terminal, download and verify JoinMarket source code.
1. Download the latest JoinMarket [release](https://github.com/JoinMarket-Org/joinmarket-clientserver/releases) and signature.

**Note:**
- At the time of writing the most recent version of JoinMarket is `v0.5.3`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -LO "https://github.com/JoinMarket-Org/joinmarket-clientserver/archive/v0.5.3.tar.gz" -O "https://github.com/JoinMarket-Org/joinmarket-clientserver/releases/download/v0.5.3/joinmarket-clientserver-0.5.3.tar.gz.asc"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   146    0   146    0     0     40      0 --:--:--  0:00:03 --:--:--    40
100 1812k    0 1812k    0     0  92094      0 --:--:--  0:00:20 --:--:--  105k
100   630    0   630    0     0   1154      0 --:--:-- --:--:-- --:--:--  1154
100   819  100   819    0     0    243      0  0:00:03  0:00:03 --:--:--   445
```
2. Receive signing key.

**Note:**
- You can verify the key fingerprint in the [release notes](https://github.com/JoinMarket-Org/joinmarket-clientserver/releases).

```
user@host:~$ gpg --recv-keys "2B6F C204 D9BF 332D 062B 461A 1410 01A1 AF77 F20B"
key 0x141001A1AF77F20B:
2 signatures not checked due to missing keys
gpg: key 0x141001A1AF77F20B: public key "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
```
3. Verify.

```
user@host:~$ gpg --verify joinmarket-clientserver-0.5.3.tar.gz.asc v0.5.3.tar.gz
gpg: Signature made Sun 03 Feb 2019 02:25:37 PM UTC
gpg:                using RSA key 0x141001A1AF77F20B
gpg: Good signature from "Adam Gibson (CODE SIGNING KEY) <ekaggata@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 2B6F C204 D9BF 332D 062B  461A 1410 01A1 AF77 F20B
```
4. Extract and enter JoinMarket directory.

```
user@host:~$ tar -C ~ -xf v0.5.3.tar.gz
user@host:~$ cd ~/joinmarket-clientserver-0.5.3
```
### B. Install JoinMarket dependencies.
1. Create python virtual environment.

```
user@host:~/joinmarket-clientserver-0.5.3$ virtualenv -p python3 jmvenv
Already using interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/user/joinmarket-clientserver-0.5.3/jmvenv/bin/python3
Also creating executable in /home/user/joinmarket-clientserver-0.5.3/jmvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
```
2. Install dependencies to virtual environment.

**Note:**
- This step, and the next optional step, will produce a lot of output and take some time. This is normal, be patient.

```
user@host:~/joinmarket-clientserver-0.5.3$ source jmvenv/bin/activate
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3$ python setupall.py --all
```
#### Optional Step: Install QT dependencies for JoinMarket GUI.
**Note:**
- You can safely skip this step if you do not intend to use the JoinMarket GUI.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3$ pip install PySide2 https://github.com/sunu/qt5reactor/archive/58410aaead2185e9917ae9cac9c50fe7b70e4a60.zip
```
3. Deactivate virtual environment and make relocatable.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3$ deactivate
user@host:~/joinmarket-clientserver-0.5.3$ virtualenv -p python3 --relocatable jmvenv
```
4. Return to home directory.

```
user@host:~/joinmarket-clientserver-0.5.3$ cd
```
### C. Relocate `joinmarket-clientserver/` directory.
1. Copy `joinmarket-clientserver/` directory to the `joinmarket` user's home directory, change ownership.

```
user@host:~$ sudo cp -r ~/joinmarket-clientserver-0.5.3/ /home/joinmarket/
user@host:~$ sudo chown -R joinmarket:nogroup /home/joinmarket/
```
2. Copy `joinmarket-clientserver/` directory to the `joinmarket` VM.

**Note:**
- Select `joinmarket` from the `dom0` pop-up.

```
user@host:~$ qvm-copy ~/joinmarket-clientserver-0.5.3/
```
### D. Start `joinmarketd` service.
```
user@host:~$ sudo systemctl start joinmarketd.service
```
## IV. Configure `bitcoind` and `joinmarketd`
### A. In a `sys-bitcoind` terminal, find out the gateway IP.
**Note:**
- Save your gateway IP (`10.137.0.50` in this example) for later to replace `<gateway-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.50
```
### B. In a `bitcoind` terminal, create RPC credentials for JoinMarket to communicate with `bitcoind`.
1. Create a random RPC username. Do not use the one shown.

**Note:**
- Save your username (`uJDzc07zxn5riJDx7N5m` in this example) for later to replace `<rpc-user>` in examples.

```
user@host:~$ head -c 15 /dev/urandom | base64
uJDzc07zxn5riJDx7N5m
```
2. Use Bitcoin's tool to create a random RPC password and config entry. Do not use the one shown.

**Notes:** 
- Save the hased password (`838c4dd74606918f1f27a5a2a52b168$9634018b87451bca05082f51b0b5b876fc72ef877bad98298e97e277abd5f90c` in this example) for later to replace `<hashed-pass>` in examples.
- Save your password (`IuziNnTsOUkonsDD3jn5WatPnFrFOMSnGUsRSUaq5Qg=` in this example) for later to replace `<rpc-pass>` in examples.
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
user@host:~$ sudo -u bitcoin kwrite /home/bitcoin/.bitcoin/bitcoin.conf
```
2. Paste the following at the bottom of the file.

**Notes:**
- Be sure not to alter any of the existing information.
- Replace `<rpc-user>` and `<hashed-pass>` with the information noted earlier.

```
# JoinMarket Auth
rpcauth=<rpc-user>:<hashed-pass>
# JoinMarket Wallet
wallet=joinmarket
```
3. Save the file.
4. Switch back to `bitcoind` terminal, restart the `bitcoind` service.

```
user@host:~$ sudo systemctl restart bitcoind.service
```

### D. Set up `qubes-rpc` for `joinmarketd`.
1. Create `joinmarketd` rpc action files.

```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:127.0.0.1:27183' > /rw/usrlocal/etc/qubes-rpc/qubes.joinmarketd-27183"
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:127.0.0.1:27184' > /rw/usrlocal/etc/qubes-rpc/qubes.joinmarketd-27184"
```
2. Fix permissions.

```
user@host:~$ sudo chmod 0644 /rw/usrlocal/etc/qubes-rpc/joinmarketd*
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
socat TCP-LISTEN:27183,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm bitcoind qubes.joinmarketd-27183" &
socat TCP-LISTEN:27184,fork,bind=127.0.0.1 EXEC:"qrexec-client-vm bitcoind qubes.joinmarketd-27184" &
```
3. Save the file.
4. Switch back to the `joinmarket` terminal, fix permissions, and execute the file.

```
user@host:~$ sudo chmod 0755 /rw/config/rc.local
user@host:~$ sudo /rw/config/rc.local
```
### B. Move copied JoinMarket directory to your home directory.
```
user@host:~$ mv ~/QubesIncoming/bitcoind/joinmarket-clientserver-0.5.3/ ~
```
### C. Source the virtual environment and enter the JoinMarket directory on boot.
**Note:**
- You should not be using the `joinmarket` VM for anything other than your JoinMarket wallet, therefore these changes should be helpful.
1. Edit the file `~/.bashrc`.

```
user@host:~$ kwrite ~/.bashrc & exit
```
2. Paste the following at the bottom of the file.

```
source /home/user/joinmarket-clientserver-0.5.3/jmvenv/bin/activate
cd /home/user/joinmarket-clientserver-0.5.3/scripts/
```
3. Save the file and open a new `joinmarket` terminal.

### D. In a `joinmarket` terminal, configure JoinMarket.
1. Generate a JoinMarket configuration file.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3/scripts$ python wallet-tool.py
Created a new `joinmarket.cfg`. Please review and adopt the settings and restart joinmarket.
```
2. Make a backup and edit the file `joinmarket.cfg`.

```
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3/scripts$ cp joinmarket.cfg joinmarket.cfg.orig
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3/scripts$ echo > joinmarket.cfg
(jmvenv) user@host:~/joinmarket-clientserver-0.5.3/scripts$ kwrite joinmarket.cfg
```
3. Paste the following.

**Notes:**
- Be sure to replace `<gateway-ip>`, `<rpc-user>`, and `<rpc-pass>` with the information noted earlier.
- For verbose desciptions of these setting, look to the original config file: `~/joinmarket-clientserver-0.5.3/scripts/joinmarket.cfg.orig`.

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
rpc_wallet_file = joinmarket

[MESSAGING:CyberguerrillaIRC]
channel = joinmarket-pit
host = 6dvj6v5imhny3anf.onion
port = 6698
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9180
usessl = true

[MESSAGING:AgoraAnarplexIRC]
channel = joinmarket-pit
host = cfyfz6afpgfeirst.onion
port = 6667
socks5 = true
socks5_host = <gateway-ip>
socks5_port = 9181
usessl = false

[MESSAGING:DarkScienceIRC]
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
merge_algorithm = default
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

## VI. Final Notes
- Once `bitcoind` has finished syncing in the `bitcoind` VM you will be able to use JoinMarket's wallet from the `joinmarket` VM. To learn more about using JoinMarket's wallet please see their [wiki](https://github.com/JoinMarket-Org/joinmarket/wiki).
