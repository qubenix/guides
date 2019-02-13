# New Guide Repo: qubes-whonix-bitcoin

- https://github.com/qubenix/qubes-whonix-bitcoin
- http://qubenixibxoyldm3l3a5fobreaydmvdweqqojllutyyi4vgtbmugvhad.onion/qubenix/qubes-whonix-bitcoin

---

# Indexed or Pruned Node?
There are three modes a Bitcoin Core node can run in: default, indexed, and pruned. For more information on these modes see [here](https://bitcoin.stackexchange.com/questions/52889/bitcoin-core-txindex-vs-default-mode-vs-pruned-mode-in-depth/52894#52894).

Default mode will not support most of the tools listed in [`indexed/`](https://github.com/qubenix/guides/blob/master/qubes-r4/whonix-14/bitcoin/indexed/), yet still requires almost the same disk space. Therefore it is not useful here.

Here are some of the differences between indexed and pruned:
## Indexed
- Stores a complete history of all transactions.
- Requires about 250G of disk space.
- Can be used as a backend for any software which needs access to the blockchain.

## Pruned
- Stores only the data concerning your addresses.
- Requires 4G of disk space.
- Can be used as a backend for some software which needs access to the blockchain.


