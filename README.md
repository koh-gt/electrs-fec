# Esplora - Electrs backend API

A block chain index engine and HTTP API written in Rust based on [romanz/electrs](https://github.com/romanz/electrs).

Used as the backend for the [Esplora block explorer](https://github.com/Blockstream/esplora) powering [blockstream.info](https://blockstream.info/).

API documentation [is available here](https://github.com/blockstream/esplora/blob/master/API.md).

Documentation for the database schema and indexing process [is available here](doc/schema.md).

Uses [rust-ferrite](https://github.com/ferritecoin/rust-ferrite) `master` branch.

### Installing & indexing

#### Install Rust, Ferrite Core (no `txindex` needed) and the `clang` and `cmake` packages, then:
```
# to install rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# if you dont have curl
sudo apt-get install curl  # do not use snap - if you did, use    sudo snap remove curl 

sudo apt install cargo

# if you hadn't already...
sudo apt install clang
sudo apt install cmake
```

```bash
$ git clone https://github.com/ferritecoin/electrs-fec && cd electrs-fec
$ cargo update   # important!
$ cargo run --release --bin electrs -- -vvvv --daemon-dir ~/.ferrite --cookie user:password --daemon-rpc-addr 127.0.0.1:9573
or alternatively $ cargo run --release --bin electrs -- -vvvv --daemon-dir ~/.ferrite

# Or for liquid:
$ cargo run --features liquid --release --bin electrs -- -vvvv --network liquid --daemon-dir ~/.liquid
```

## Known issues
### Build Errors
#### No method named `emit_rerun_if_env_changed`
```
error[E0599]: no method named `emit_rerun_if_env_changed` found for struct `Build` in the current scope
   --> /home/ubuntu/.cargo/registry/src/index.crates.io-6f17d22bba15001f/blake3-1.5.3/build.rs:104:11
    |
104 |     build.emit_rerun_if_env_changed(false);
    |           ^^^^^^^^^^^^^^^^^^^^^^^^^ method not found in `Build`

For more information about this error, try `rustc --explain E0599`.
error: could not compile `blake3` (build script) due to previous error
warning: build failed, waiting for other jobs to finish...
```
```bash
# Open your Cargo.toml file and ensure the cc dependency is set to a version that includes the emit_rerun_if_env_changed method:
[build-dependencies]
cc = "1.0.72"
# then
$ cargo update
# and retry
$ cargo run --features liquid --release --bin electrs -- -vvvv --network liquid --daemon-dir ~/.liquid
```
### RPC Errors
#### failed to read cookie
```
WARN - reconnecting to ferrited: failed to read cookie from "/home/ubuntu/.ferrite/.cookie"
```
```bash
# Assuming your RPC username is "user" and RPC password is "password" without the two enclosing double quotation marks.

$ cargo run --release --bin electrs -- -vvvv --daemon-dir ~/.ferrite --cookie user:password --daemon-rpc-addr 127.0.0.1:9573

```
#### No reply from daemon
```
WARN - reconnecting to ferrited: no reply from daemon
# from debug.log
ThreadRPCServer incorrect password attempt from 127.0.0.1:56148
```

See [electrs's original documentation](https://github.com/romanz/electrs/blob/master/doc/usage.md) for more detailed instructions.
Note that our indexes are incompatible with electrs's and has to be created separately.

The indexes require 610GB of storage after running compaction (as of June 2020), but you'll need to have
free space of about double that available during the index compaction process.
Creating the indexes should take a few hours on a beefy machine with SSD.

To deploy with Docker, follow the [instructions here](https://github.com/Blockstream/esplora#how-to-build-the-docker-image).

### Light mode

For personal or low-volume use, you may set `--lightmode` to reduce disk storage requirements
by roughly 50% at the cost of slower and more expensive lookups.

With this option set, raw transactions and metadata associated with blocks will not be kept in rocksdb
(the `T`, `X` and `M` indexes),
but instead queried from litecoind on demand.

### Notable changes from Electrs:

- HTTP REST API in addition to the Electrum JSON-RPC protocol, with extended transaction information
  (previous outputs, spending transactions, script asm and more).

- Extended indexes and database storage for improved performance under high load:

  - A full transaction store mapping txids to raw transactions is kept in the database under the prefix `t`.
  - An index of all spendable transaction outputs is kept under the prefix `O`.
  - An index of all addresses (encoded as string) is kept under the prefix `a` to enable by-prefix address search.
  - A map of blockhash to txids is kept in the database under the prefix `X`.
  - Block stats metadata (number of transactions, size and weight) is kept in the database under the prefix `M`.

  With these new indexes, litecoind is no longer queried to serve user requests and is only polled
  periodically for new blocks and for syncing the mempool.

- Support for Liquid and other Elements-based networks, including CT, peg-in/out and multi-asset.
  (requires enabling the `liquid` feature flag using `--features liquid`)

### CLI options

In addition to electrs's original configuration options, a few new options are also available:

- `--http-addr <addr:port>` - HTTP server address/port to listen on (default: `127.0.0.1:3000`).
- `--lightmode` - enable light mode (see above)
- `--cors <origins>` - origins allowed to make cross-site request (optional, defaults to none).
- `--address-search` - enables the by-prefix address search index.
- `--index-unspendables` - enables indexing of provably unspendable outputs.
- `--utxos-limit <num>` - maximum number of utxos to return per address.
- `--electrum-txs-limit <num>` - maximum number of txs to return per address in the electrum server (does not apply for the http api).
- `--electrum-banner <text>` - welcome banner text for electrum server.

Additional options with the `liquid` feature:
- `--parent-network <network>` - the parent network this chain is pegged to.

Additional options with the `electrum-discovery` feature:
- `--electrum-hosts <json>` - a json map of the public hosts where the electrum server is reachable, in the [`server.features` format](https://electrumx.readthedocs.io/en/latest/protocol-methods.html#server.features).
- `--electrum-announce` - announce the electrum server on the electrum p2p server discovery network.

See `$ cargo run --release --bin electrs -- --help` for the full list of options.

## License

MIT
