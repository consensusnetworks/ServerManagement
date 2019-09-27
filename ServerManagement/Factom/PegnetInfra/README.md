# Pegnet server setup, config, and operation.

## Prerequisites 

First you need to install factomd and walletd. Go to https://github.com/FactomProject/distribution and get the most recent
distributions. 
```
wget https://github.com/FactomProject/distribution/releases/download/v6.3.3/factom-amd64.deb
wget https://github.com/FactomProject/distribution/releases/download/v6.3.1/enterprise-wallet-linux.zip
sudo dpkg -i 'factom-amd64'
sudo dpkg -i 'enterprise-wallet-linux'
```
Sometimes installing this way can be finicky and it will error out saying out don't have a dependency you need. if This happens
simply install whatever dependency it says you lack using `sudo apt-get DEPENDENCY_NAME` and once it stops installing you should be good!

Create a .pegnet folder inside your home directory. Copy the config/defaultconfig.ini file there.

```
mkdir ~/.pegnet
wget https://raw.githubusercontent.com/pegnet/pegnet/master/config/defaultconfig.ini -P ~/.pegnet/
```

### API Keys and Config File Setup

Pegnet uses several apis to act as oracles for the network. At the bare minimum you need the currency layer API found at  https://currencylayer.com. Once 
You have signed up for your free api, open the config file using `sudo nano ~/.pegnet/defaultconfig.ini` and modify the below
section with your API Key(s), Desired Identity Chain, EC_ADDRESS, & FCT_ADDRESS:
```
# The targeted cutoff. If our difficulty will land us in the top 300 (estimated), we will submit our OPR.
# <=0 will disable this check.
  SubmissionCutOff=200
  Protocol=PegNet
  Network=MainNet

  # For LOCAL network testing, EC private key is
  # Es2XT3jSxi1xqrDvS5JERM3W3jh1awRHuyoahn3hbQLyfEi1jvbq EC3TsJHUs8bzbbVnratBafub6toRYdgzgbR7kWwCW4tqbmyySRmg
  ECAddress=EC3g5ZqbaynhBsV9sm6BXxakhBEoab17xSmmnLC3rA4Z8wMwabiX

  # For LOCAL network testing, we have a public FCT address for your use.
  #
  # FCT private key
  #     Fs3E9gV6DXsYzf7Fqx1fVBQPQXV695eP3k5XbmHEZVRLkMdD9qCK
  # FCT public key
  #     FA2jK2HcLnRdS94dEcU27rF3meoJfpUcZPSinpb7AwQvPRY6RL1Q

  # This is your address.  Generate a FCT address and put it here, so if you win, you will be rewarded!
  FCTAddress=YOUR_FCT_ADDRESS
  CoinbaseAddress=YOUR_COINBASE_ADDRESS

  # The IdentityChain is to be the name of the miner's Identity.  To be used
  # when we integrate DIDs with the PegNet.  Now it acts like a public memo
  # field on your OPR records.
  IdentityChain=YOUR_IDENTITY_CHAIN
[Oracle]

  # Must get a key from here https://apilayer.com/
  APILayerKey=YOUR_API_KEY
  # Must get an api key here https://openexchangerates.org/
  OpenExchangeRatesKey=OPTIONAL_KEY_CHANGE_IF_DESIRED
  # Must get an api key here https://coinmarketcap.com/api/
  CoinMarketCapKey=OPTIONAL_KEY_CHANGE_IF_DESIRED
  # Must get an api key here https://1forge.com/forex-data-api
  1ForgeKey=OPTIONAL_KEY_CHANGE_IF_DESIRED
```

Once complete, pull the latest binaries for Pegnet and initialize the protocol. ***NOTE*** Check the version you are using.
Current version as of writing this is 0.2.2, but updates occur ***OFTEN*** so please check first.
```
wget https://github.com/pegnet/pegnet/releases/download/v0.2.2/pegnet_linux_amd64
chmod +x pegnet_linux_amd64
./pegnet_linux_amd64
```
With Pegnet running, all you need to do is now daemonize it. To do so, create a systemd service 

```
sudo nano /etc/systemd/system/pegnet.service
```
and then paste the following:
```
[Unit]
Description=Pegnet Miner
After=network-online.target

[Service]
ExecStart=/home/sysadmin/.pegnet/pegnet_linux_amd64 --log debug

SyslogIdentifier=pegnet
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```
restart the daemon:
```
systemctl daemon-reload
systemctl start pegnet
```
Check that everything is running Properly:
```
systemctl status pegnet
```
***NOTE***: Sometimes the pegnet binaries are finicky and systemd will not want to execute them. If this occurs you should
be able to run the software with Linux Screen. To do so:
```
screen
./pegnet_linux_amd64
```
Then press ctrl + ad to detatch from the screen, and it should remain running as a backround process.

## Updating
Simply stop the pegnet service, delete the pegnet binaries, grab the newest binaries, execute them and repeat. The code below
assumes you are using systemd, if this is not the case simply attatch to the scrren session you were in, follow the same steps
and then detatch once more.
```
systemctl stop pegnet
cd ~/.pegnet
sudo rm -R pegnet
wget http://PATH_TO_LATEST_BINARIES
chmod +x pegnet_linux_amd64
./pegnet_linux_amd64
```
Once synced, stop using ctrl +C, then restart the pegnet service.


