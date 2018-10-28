# Indexed or Pruned Node?

[Here](https://bitcoin.stackexchange.com/questions/52889/bitcoin-core-txindex-vs-default-mode-vs-pruned-mode-in-depth/52894#52894) is an explanation of the three modes: default, indexed, and pruned.

I don't see any point in running default mode as it only saves 17G of disk space from an indexed node, and you have to reindex anyway if you want to connect other tools. This is why I only make guides for indexed or pruned.

## Indexed

+ Stores a complete history of Bitcoin's blockchain.
+ Requires 250G of disk space.
+ Versatile use cases.

## Pruned

+ Stores only the data concerning your addresses.
+ Requires 4G of disk space.
