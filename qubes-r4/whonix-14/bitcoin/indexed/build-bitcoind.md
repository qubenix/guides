# Qubes 4 & Whonix 14: Building a Bitcoin Core Full Node
Build a [Bitcoin Core](https://github.com/bitcoin/bitcoin) full node from source code and configure it to communicate only over Tor, prefer hidden services, utilize stream isolation, index all transactions, and use ephemeral hidden services to serve peers.

## What is Bitcoin Core?
The server daemon for the Bitcoin distributed cryptocurrency (`bitcoind`), command line tools (`bitcoin-cli`), and a gui wallet (`bitcoin-qt`). These tools can be used to observe and interact with Bitcoin's blockchain.

## Why Do This?
Bitcoin Core is a "full node" implementation, meaning it will verify that all incoming transactions and blocks are following Bitcoin's rules. This allows you to validate transactions without trusting third parties. Furthermore, an indexed node can be used as a backend for other software which needs access to the blockchain (block explorers, [c-lightning](https://github.com/ElementsProject/lightning), [electrum personal server](https://github.com/chris-belcher/electrum-personal-server), [joinmarket](https://github.com/JoinMarket-Org/joinmarket-clientserver), [lnd](https://github.com/LightningNetwork/lnd), etc.).

## I. Set Up Dom0

### A. In a `dom0` terminal, clone a Whonix TemplateVM.

1. In a `dom0` terminal, clone a Whonix workstation TemplateVM and launch a terminal.

```
[user@dom0 ~]$ qvm-clone whonix-ws-14 whonix-ws-14-bitcoin
```

### B. Create an AppVM from the new Whonix TemplateVM.

1. Create the VM for running `bitcoind` using a Whonix gateway for networking.

**Note:** You must pick some label color for your VMs upon creation, it does not have to match what is shown here. `sys-whonix` is the default name of the Whonix gateway.

```
[user@dom0 ~]$ qvm-create --label red --prop netvm='sys-whonix' --template whonix-ws-14-bitcoin bitcoind
```

2. Increase private volume size and enable `bitcoind` service.

```
[user@dom0 ~]$ qvm-volume resize bitcoind:private 250G
[user@dom0 ~]$ qvm-service --enable bitcoind bitcoind
```

## II. Set Up TemplateVM.

### A. In the `whonix-ws-14-bitcoin` terminal, update and install dependencies.

```
user@host:~$ sudo apt-get update && sudo apt-get install -y automake autotools-dev bsdmainutils build-essential git libboost-chrono-dev libboost-filesystem-dev libboost-system-dev libboost-test-dev libboost-thread-dev libevent-dev libprotobuf-dev libqrencode-dev libqt5core5a libqt5dbus5 libqt5gui5 libssl-dev libtool libzmq3-dev pkg-config protobuf-compiler python3 qttools5-dev qttools5-dev-tools
```

### B. Create system user.

```
user@host:~$ sudo adduser --group --system bitcoind
Adding system user `bitcoind' (UID 116) ...
Adding new group `bitcoind' (GID 122) ...
Adding new user `bitcoind' (UID 116) with group `bitcoind' ...
Creating home directory `/home/bitcoind' ...
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
PIDFile=/home/bitcoind/.bitcoin/bitcoind.pid
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

3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ sudo chmod 0644 /lib/systemd/system/bitcoind.service
```

4. Enable the service.

```
user@host:~$ sudo systemctl enable bitcoind.service
Created symlink /etc/systemd/system/multi-user.target.wants/bitcoind.service → /lib/systemd/system/bitcoind.service.
```

### D. Shutdown TemplateVM.

```
user@host:~$ sudo shutdown now
```

## III. Set Up Gateway.

### A. In a `sys-whonix` terminal, find out the gateway IP.

**Note:** save your gateway IP for later to replace `<gateway-ip>` in examples.

```
user@host:~$ qubesdb-read /qubes-ip
10.137.0.xx
```

### B. Configure `onion-grater`.

1. Create persistent directory and `onion-grater` profile.

```
user@host:~$ sudo mkdir -m 0755 /usr/local/etc/onion-grater-merger.d
user@host:~$ sudo kwrite /usr/local/etc/onion-grater-merger.d/40_bitcoind.yml
```

2. Paste the following.

```
---
- exe-paths:
    - '*'
  users:
    - '*'
  hosts:
    - '*'
  commands:
    ADD_ONION:
      ## {{{ Mainnet onion service.
      - pattern:     'NEW:(\S+) Port=8333,127.0.0.1:8333'
        replacement: 'NEW:{} Port=8333,{client-address}:8333 Flags=DiscardPK'
      ## Testnet onion service.
      - pattern:     'NEW:(\S+) Port=18333,127.0.0.1:18333'
        replacement: 'NEW:{} Port=18333,{client-address}:18333 Flags=DiscardPK'
      ## Needed to make services ephemeral.
      - pattern:     '250-PrivateKey=(\S+):\S+'
        replacement: '250-PrivateKey={}:redacted'
      ## }}}
```

3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ sudo chmod 0644 /usr/local/etc/onion-grater-merger.d/40_bitcoind.yml
```

4. Restart `onion-grater` service.

```
user@host:~$ sudo systemctl restart onion-grater.service
```

## IV. Install Bitcoin

### A. In a `bitcoind` terminal, download and verify the Bitcoin source code.

1. Clone the repository, receive signing keys, and verify source.

**Note:** at the time of writing the branch of the current release is `0.17`, modify the following steps accordingly if the version has changed.

```
user@host:~$ git clone --branch 0.17 https://github.com/bitcoin/bitcoin
Cloning into 'bitcoin'...
remote: Enumerating objects: 12, done.
remote: Counting objects: 100% (12/12), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 124912 (delta 6), reused 3 (delta 3), pack-reused 124900
Receiving objects: 100% (124912/124912), 111.85 MiB | 537.00 KiB/s, done.
Resolving deltas: 100% (87124/87124), done.
user@host:~$ gpg --recv-keys AC6626172E00A82CFFAE8972A636E97631F767E0
gpg: keybox '/home/user/.gnupg/pubring.kbx' created
gpg: /home/user/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x860FEB804E669320: public key "Pieter Wuille <pieter.wuille@gmail.com>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1
user@host:~$ cd bitcoin/
user@host:~/bitcoin$ git verify-commit HEAD
gpg: Signature made Sat 20 Oct 2018 02:04:39 AM UTC
gpg:                using RSA key AC6626172E00A82CFFAE8972A636E97631F767E0
gpg: Good signature from "Pieter Wuille <pieter.wuille@gmail.com>" [unknown]
gpg:                 aka "[jpeg image of size 5996]" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 133E AC17 9436 F14A 5CF1  B794 860F EB80 4E66 9320
     Subkey fingerprint: AC66 2617 2E00 A82C FFAE  8972 A636 E976 31F7 67E0
```

### B. Build Berkeley DB and Bitcoin.

**Note:** each of these next steps will produce a lot of output in the terminal. This is normal, be patient.

1. Build Berkeley DB using the provided script.

```
user@host:~/bitcoin$ ./contrib/install_db4.sh `pwd`
```

2. Build and install Bitcoin Core.

```
user@host:~/bitcoin$ export BDB_PREFIX='/home/user/bitcoin/db4'; ./autogen.sh && ./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" BDB_CFLAGS="-I${BDB_PREFIX}/include" --without-miniupnpc && make && sudo make install
```

## V. Set Up Bitcoin.

### A. In a `bitcoind` terminal, configure Bitcoin.

1. Create Bitcoin's data directory and configuration file.

```
user@host:~$ sudo mkdir -m 0700 /home/bitcoind/.bitcoin
user@host:~$ sudo kwrite /home/bitcoind/.bitcoin/bitcoin.conf
```

2. Paste the following.

**Note:** be sure to replace `<gateway-ip>` with the information noted earlier.

```
daemon=1
listen=1
onlynet=onion
proxy=<gateway-ip>:9111
proxyrandomize=1
server=1
txindex=1
```

3. Save the file, switch back to the terminal, and fix permissions.

```
user@host:~$ sudo chmod 0600 /home/bitcoind/.bitcoin/bitcoin.conf
user@host:~$ sudo chown -R bitcoind:bitcoind /home/bitcoind/.bitcoin
```

### B. Open p2p port.

1. Make persistent directory, configure firewall, and fix permissions.

```
user@host:~$ sudo mkdir -m 0755 /rw/config/whonix_firewall.d
user@host:~$ sudo sh -c 'echo "EXTERNAL_OPEN_PORTS+=\" 8333 \"" > /rw/config/whonix_firewall.d/50_user.conf'
user@host:~$ sudo chmod 0644 /rw/config/whonix_firewall.d/50_user.conf
```

2. Restart firewall service.

```
user@host:~$ sudo systemctl restart whonix-firewall.service
```

## VI. Create Communication Channel

**Note:** this only creates the possibility for other VMs to communicate with `bitcoind`, it does not yet give them permission.

### A. In a `bitcoind` terminal, create `qubes-rpc` action files.

1. Create persistent directory for `qrexec` action files.

```
user@host:~$ sudo mkdir -m 0755 /rw/usrlocal/etc/qubes-rpc
```

2. Create `bitcoind` action file.

```
user@host:~$ sudo sh -c "echo 'socat STDIO TCP:localhost:8332' > /rw/usrlocal/etc/qubes-rpc/qubes.bitcoind"
```
