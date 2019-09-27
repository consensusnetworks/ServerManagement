# Polkadot/Kusama Server Setup, Configuration, & Operation

## Prerequisites

First you must install Rust
```
curl https://sh.rustup.rs -sSf | sh
sudo apt install make clang pkg-config libssl-dev
rustup update
```

***NOTE***: If you are running a testnet node on ***PoC-4***  you must also run the following commands as well:
```
rustup toolchain install nightly-2019-07-14
rustup default nightly-2019-07-14
rustup target add wasm32-unknown-unknown --toolchain nightly-2019-07-14
```

## Installing polkadot/kusama

In the below steps, you will clone the paritytech repo, checkout the version you need and install polkadot/kusama. Pay very
close attention to the version you are checking out. As of Writing this, Polkadot PoC-4 is on v0.4 and Kusama is on v0.6.0. 
The directions below will assume that you are setting up a polkadot node, but the process is identical for Kusama other than
just checking out a certain version. ***NOTE***: These versions change often, so please check you are using the correct version via Discord
or Riot 

```
git clone https://github.com/paritytech/polkadot.git
cd polkadot
cargo clean
git checkout v0.4
./scripts/init.sh
cargo build --release
```
***NOTE*** The latest Kusama version would not build correctly as of writing this (092719). If this is the case, you can
simply download the binaries and install as shown below:
```
wget https://releases.parity.io/polkadot/x86_64-debian:stretch/v0.4/polkadot
chmod +x polkadot
./polkadot
```
***NOTE***: Again, check that you are grabbing the right version of polkadot if you are doing it this way. This will actually 
install and start synching the chain so if you do it as shown above using the binaries, you simply need to create an appropriate
systemd service and you will be good to go!

## Synchronizing the Chain

To start synchonizing the chain simply run
```
./target/release/polkadot
```
The chain should begin syncing and you should be good to go!
To daemonize the service create a systemd service file by running `sudo nano /etc/systemd/system/polkadot.service`
Then paste the code below:
```
[Unit]
Description=Polkadot Node
After=network-online.target

[Service]
ExecStart=/home/sysadmin/polkadot/target/release/polkadot 
#If you used wget your path will be /home/sysadmin/polkadot (delete this line)

SyslogIdentifier=polkadot
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```
Restart the daemon:
```
systemctl daemon-reload
systemctl start polkadot
```
Check that everything is running Properly:
```
systemctl status polkadot
