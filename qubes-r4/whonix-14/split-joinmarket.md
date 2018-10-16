# Qubes 4 & Whonix 14: Split JoinMarket Wallet
Create a two VM system for a fully functional JoinMarket wallet in an offline VM. The `joinmarketd` daemon and pruned `bitcoind` node run in a separate Whonix VM, communicate only over Tor, prefer hidden services, and use stream isolation.

## What is JoinMarket?
Joinmarket is a decentralized, open source, and trustless market for Bitcoin privacy using [coinjoin](https://en.bitcoin.it/wiki/CoinJoin). Anyone holding Bitcoin can offer coinjoins for a fee, and anyone can pay a fee to have their transactions obfuscated.

There is a detailed explanation of the concept by the creator [here](https://bitcointalk.org/index.php?topic=919116.0), and a descriptive infographic [here](https://imgur.com/C6w0Pgf).

## How Safe is This?
The benefit of this setup is that your wallet VM is essentially cold storage, yet retains all the functionality of an internet connected JoinMarket wallet (send payment, yield generator, etc).

The only way a remote attacker can compromise this system is to successfully exploit one of your internet connected VMs and then use a Qubes/Xen 0-day to escape that VM.

## I. Create VMs

### A. Clone Whonix TemplateVM and install packages.

1. In a `dom0` terminal, clone a Whonix workstation TemplateVM and launch a terminal.

```
[user@dom0 ~]$ qvm-clone whonix-ws-14 whonix-ws-jm-14
[user@dom0 ~]$ qvm-run whonix-ws-jm-14 konsole
```

2. In the `whonix-ws-jm-14` terminal, install packages and shutdown VM.

```
user@host:~$ sudo apt-get update && sudo apt-get install automake build-essential curl git libffi-dev libsecp256k1-dev libsodium-dev libtool pkg-config python-dev python-pip python-sip python-virtualenv -y
user@host:~$ sudo shutdown now
```

### B. In a `dom0` terminal, create two AppVMs from the new Whonix TemplateVM.

1. Create the VM for `bitcoind` and `joinmarketd` using a Whonix gateway for networking, and increase the private volume size.

**Note:** You must pick some label color for your VMs upon creation. The color does not have to match what is shown in these examples.

```
[user@dom0 ~]$ qvm-create --label red --prop netvm='sys-whonix' --template whonix-ws-jm-14 jm-bitcoind
[user@dom0 ~]$ qvm-volume resize jm-bitcoind:private 20G
```

2. Create the VM for JoinMarket's wallet with no networking.

```
[user@dom0 ~]$ qvm-create --label black --prop netvm='' --template whonix-ws-jm-14 jm-wallet
```

## II. Install Bitcoin and JoinMarket

### A. In the `jm-bitcoind` terminal, install Bitcoin, JoinMarket, and dependencies.

1. Download, verify, and install the Linux 64 bit version of [Bitcoin Core](https://bitcoincore.org/en/download/).

**Note:** at the time of writing the most recent version of Bitcoin Core is `0.17.0`, modify the following steps accordingly if the version has changed.

```
user@host:~$ curl -O "https://bitcoincore.org/bin/bitcoin-core-0.17.0/bitcoin-0.17.0-x86_64-linux-gnu.tar.gz" -O "https://bitcoincore.org/bin/bitcoin-core-0.17.0/SHA256SUMS.asc"
user@host:~$ gpg --recv-keys 01EA5486DE18A882D4C2684590C8019E36C2E964
gpg: keybox '/home/user/.gnupg/pubring.kbx' created
gpg: /home/user/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x90C8019E36C2E964: public key "Wladimir J. van der Laan (Bitcoin Core binary release signing key) <laanwj@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
user@host:~$ gpg --verify SHA256SUMS.asc 
gpg: Signature made Wed 03 Oct 2018 08:53:25 AM UTC
gpg:                using RSA key 0x90C8019E36C2E964
gpg: Good signature from "Wladimir J. van der Laan (Bitcoin Core binary release signing key) <laanwj@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 01EA 5486 DE18 A882 D4C2  6845 90C8 019E 36C2 E964
user@host:~$ cat SHA256SUMS.asc | grep x86_64 | shasum -c
bitcoin-0.17.0-x86_64-linux-gnu.tar.gz: OK
user@host:~$ tar xf bitcoin-0.17.0-x86_64-linux-gnu.tar.gz
user@host:~$ sudo install -g staff -m 0755 -o root bitcoin-0.17.0/bin/bitcoin* -t /usr/local/bin/
```

2. Download and verify [JoinMarket](https://github.com/JoinMarket-Org/joinmarket-clientserver).

**Note:** at the time of writing the most recent version of JoinMarket is `0.3.5`, modify the following steps accordingly if the version has changed.

```
user@host:~$ git clone https://github.com/JoinMarket-Org/joinmarket-clientserver ~/joinmarket-clientserver
Cloning into '/home/user/joinmarket-clientserver'...
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 3203 (delta 3), reused 4 (delta 0), pack-reused 3192
Receiving objects: 100% (3203/3203), 2.86 MiB | 19.00 KiB/s, done.
Resolving deltas: 100% (2101/2101), done.
user@host:~$ gpg --recv-keys 46689728A9F64B391FA871B7B3AE09F1E9A3197A
gpg: key 0xB3AE09F1E9A3197A: public key "Adam Gibson <ekaggata@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
user@host:~$ cd ~/joinmarket-clientserver/
user@host:~/joinmarket-clientserver$ git checkout -q v0.3.5
user@host:~/joinmarket-clientserver$ git verify-commit HEAD
gpg: Signature made Fri 03 Aug 2018 09:39:47 PM UTC
gpg:                using RSA key B3AE09F1E9A3197A
gpg: Good signature from "Adam Gibson <ekaggata@gmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 4668 9728 A9F6 4B39 1FA8  71B7 B3AE 09F1 E9A3 197A
```

3. Create python virtual environment and install packages to it.

```
user@host:~/joinmarket-clientserver$ virtualenv jmvenv
Running virtualenv with interpreter /usr/bin/python2
New python executable in /home/user/joinmarket-clientserver/jmvenv/bin/python2
Also creating executable in /home/user/joinmarket-clientserver/jmvenv/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
user@host:~/joinmarket-clientserver$ source jmvenv/bin/activate
(jmvenv) user@host:~/joinmarket-clientserver$ python setupall.py --client-bitcoin && python setupall.py --daemon
(jmvenv) user@host:~/joinmarket-clientserver$ deactivate
```

4. Copy `joinmarket-clientserver/` directory to the `jm-wallet` VM.

**Note:** select `jm-wallet` from the `dom0` pop-up.

```
user@host:~/joinmarket-clientserver$ qvm-copy ~/joinmarket-clientserver/
```

## III. Configure `bitcoind` and `joinmarketd`

### A. In a `sys-whonix` terminal, find out the gateway IP.

**Note:** save your gateway IP for later to replace `<gateway-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.xx
```

### B. In a `jm-bitcoind` terminal, create RPC credentials for JoinMarket to communicate with `bitcoind`.

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
deprecatedrpc=accounts
discover=0
listen=0
listenonion=0
onlynet=onion
proxy=<gateway-ip>:9111
prune=550
rpcuser=<rpc-user>
rpcpassword=<rpc-pass>
server=1
wallet=jm-wallet.dat
```

3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ chmod 0600 ~/.bitcoin/bitcoin.conf
```

### D. Use `systemd` to keep `bitcoind` always running.

1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /rw/config/bitcoind.service
```

2. Paste the following.

```
[Unit]
Description=Bitcoin's distributed currency daemon

[Service]
User=user
Group=user

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

### E. Use `systemd` to keep `joinmarketd` always running.

1. Create `systemd` service file.

```
user@host:~$ sudo kwrite /rw/config/joinmarketd.service
```

2. Paste the following.

```
[Unit]
Description=JoinMarket's server daemon

[Service]
User=user
Group=user

Type=idle
WorkingDirectory=/home/user/joinmarket-clientserver
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
user@host:~$ sudo chmod 0644 /rw/config/bitcoind.service /rw/config/joinmarketd.service
```

### F. Start the services on boot.

1. Edit the file `/rw/config/rc.local`.

```
user@host:~$ sudo kwrite /rw/config/rc.local
```

2. Paste the following at the bottom of the file.

```
cp /rw/config/bitcoind.service /rw/config/joinmarketd.service -t /etc/systemd/system/multi-user.target.wants/
systemctl daemon-reload
systemctl start bitcoind.service
systemctl start joinmarketd.service
```

3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ sudo chmod 0755 /rw/config/rc.local
```

## IV. Create Communication Channels for Wallet VM

### A. In a `jm-bitcoind` terminal, create `qubes-rpc` action files.

1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```

2. Create `bitcoind` action file.

```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:localhost:8332' > /rw/usrlocal/etc/qubes-rpc/qubes.bitcoind"
```

3. Create `joinmarketd` action file.

```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:localhost:27183' > /rw/usrlocal/etc/qubes-rpc/qubes.joinmarketd"
```

### B. In a `dom0` terminal, allow `jm-wallet` access to action files.

```
[user@dom0 ~]$ echo 'jm-wallet jm-bitcoind allow' | sudo tee /etc/qubes-rpc/policy/qubes.{bitcoind,joinmarketd} > /dev/null
```

## V. Configure JoinMarket Wallet VM.

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

**Note:** the directory must be moved to your home dir, otherwise the virtual environment will not work.

```
user@host:~$ mv ~/QubesIncoming/jm-bitcoind/joinmarket-clientserver/ ~
```

2. Edit the file `~/.bashrc`.

**Note:** you should not be using the `jm-wallet` VM for anything other than your JoinMarket wallet, therefore these changes should be helpful.

```
user@host:~$ kwrite ~/.bashrc & exit
```

3. Paste the following at the bottom of the file.

```
source /home/user/joinmarket-clientserver/jmvenv/bin/activate
cd /home/user/joinmarket-clientserver/scripts/
```

4. Save the file and open a new `jm-wallet` terminal.

### C. In a `jm-wallet` terminal, configure JoinMarket.

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
daemon_host = localhost
use_ssl = false

[BLOCKCHAIN]
blockchain_source = bitcoin-rpc
network = mainnet
rpc_host = 127.0.0.1
rpc_port = 8332
rpc_user = <rpc-user>
rpc_password = <rpc-pass>
rpc_wallet_file = jm-wallet.dat

[MESSAGING]
channel = joinmarket-pit, joinmarket-pit
port = 6698, 6667 
usessl = true, false
socks5 = true, true
socks5_host = <gateway-ip>, <gateway-ip>
socks5_port = 9112, 9113
host = 6dvj6v5imhny3anf.onion, cfyfz6afpgfeirst.onion

[LOGGING]
console_log_level = INFO

[TIMEOUT]
maker_timeout_sec = 60
unconfirm_timeout_sec = 90
confirm_timeout_hours = 6

[POLICY]
segwit=true
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

## VII. Possible Improvements

- Create system users in TemplateVM for running `bitcoind` and `joinmarketd`.

- Enable `systemd` units in TemplateVM with `qubes-service` condition path.

- Enable AppArmor, create profiles.

- `onion-grater` profile for `bitcoind` and remove `listenonion=0`.

- [VM hardening](https://github.com/tasket/Qubes-VM-hardening).

## VIII. To Do

- Watch for new release of JM to remove `deprecatedrpc=accounts` from bitcoin.conf.

- Watch [here](https://github.com/bitcoin/bitcoin/issues/14292) for rpcauth.py to be added to Bitcoin's bins and replace `rpc{user,pass}=` with `rpcauth=` in bitcoin.conf.
