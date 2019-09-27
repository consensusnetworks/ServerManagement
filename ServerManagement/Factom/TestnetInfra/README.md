# Factom Testnet server setup, configuration, and operating instructions

## Prereqs and Config

*Supplementa lLinks*
http://www.factom-testnet.com/Introduction

### The server needs to be configured to listen to the following ports for the following IP addresses 

TCP port 2376 open to 54.171.68.124 for secure Docker engine communication

2222 to 54.171.68.124, which is the SSH port used by the ssh container

8088 to 54.171.68.124, the factomd API port

8090 to 0.0.0.0, the factomd Control panel, Keeping this open to the world is beneficial on testnet for debugging purposes

8110 to 0.0.0.0, the factomd testnet port

On Ubuntu this is done as follows:
```
sudo ufw enable
sudo ufw allow from 54.171.68.124 to any port 2376
sudo ufw allow from 54.171.68.124 to any port 2222
sudo ufw allow from 54.171.68.124 to any port 8088
sudo ufw allow from 0.0.0.0/0 to any port 8090
sudo ufw allow from 0.0.0.0/0 to any port 8110
```

Update and install required packages:
```
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install curl git apt-transport-https ca-certificates curl software-properties-common -y
```
Add the docker-ce repository and install:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get install docker-ce
```
Add docker privileges to the current user:
```
sudo usermod -aG docker $USER
```
***Configuring Docker***

*Pay attention to the directory*
Make sure you store the Docker Swarm mainnet key and certificate on your system. 
```
sudo mkdir -p /etc/docker
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-testnet-toolkit/master/tls/cert.pem -O /etc/docker/factom-mainnet-cert.pem
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-testnet-toolkit/master/tls/key.pem -O /etc/docker/factom-mainnet-key.pem
sudo chmod 644 /etc/docker/factom-mainnet-cert.pem
sudo chmod 440 /etc/docker/factom-mainnet-key.pem
sudo chgrp docker /etc/docker/*.pem
```

Create a json file, this will help start factomd through docker:

```sudo nano /etc/docker/daemon.json```
Paste this in the file (ensure path is correct!)
```
{
  "tls": true,
  "tlscert": "/etc/docker/factom-mainnet-cert.pem",
  "tlskey": "/etc/docker/factom-mainnet-key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock", "fd://"]
}
```

Create a systemd service to ensure docker/factomd autmatically starts
```
sudo systemctl edit docker.service
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```
then reload
```	
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```
If you need to restart
```
systemctl stop docker.service
systemctl stop docker.socket
systemctl daemon-reload
systemctl start docker.service
systemctl status docker.service docker.socket
```

moving old database
```
sudo cp -r /data/var/lib/docker/volumes/factom_database/_data /var/lib/docker/volumes/factom_database/_data
```
## Starting the Node

Create the FactomD volumes to hold your identity keys and the database 
```
docker volume create factom_database
docker volume create factom_keys
```
Need to create factom.conf file, this will initialize the factom docker instance when it starts:

This line pulls the .config file
```
sudo curl https://raw.githubusercontent.com/FactomProject/factomd-testnet-toolkit/master/factomd.conf.EXAMPLE -o /var/lib/docker/volumes/factom_keys/_data/factomd.conf

```

#Join the Factom Testnet Swarm:
```
docker swarm join --token SWMTKN-1-0bv5pj6ne5sabqnt094shexfj6qdxjpuzs0dpigckrsqmjh0ro-87wmh7jsut6ngmn819ebsqk3m 54.171.68.124:2377
```

*Start the FactomD container (pay attention to the factomd version, as of 11/20/18 is v6.0.1-alpine)*
```
docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8110:8110" -l "name=factomd" factominc/factomd:v6.1.0-rc1-alpine -broadcastnum=16 -network=CUSTOM -customnet=fct_community_test -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
```

The below command will give you your node ID so that other peers can find you in the network/
```
docker info | grep NodeID
```

***This will remove factom_database if you need to start over rebuilding the ledger***
```
sudo docker volume rm -f factom_database
```

## Updating Factomd (need to pay attention to ***testnet version***):
```
sudo docker stop factomd
sudo docker rm factomd
sudo docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8110:8110" -l "name=factomd" factominc/factomd:v6.2.3-rc3-alpine -broadcastnum=16 -network=CUSTOM -customnet=fct_community_test -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
```

## Setting Up Server Identity

Factom Servers have an identity associated with them. This is the most difficult and tedious part of installation. ***Before
You Begin you will need to have an offline computer and USB drive handy.***

### Install Factom wallet and factomd-cli

Go to: https://github.com/FactomProject/distribution
to get the latest releases of both factom-cli and walletd

to download dependency files:
```
sudo wget https://path/to/latest releases.dep
```
Once downloaded, for both files:
```
sudo dpkg -i 'current.factom.release.dep'
```
Create a systemd service to automatically start the wallet when the node is initialized, make sure factom-cli is unpacked first:
```
sudo nano /etc/systemd/system/factom-walletd.service

#insert this code and save/close the file
[Unit]
Description=Run the Factom Wallet service
Documentation=https://github.com/FactomProject/factom-walletd
After=network-online.target

[Service]
User=factom-walletd #This assumes user, if using root, delete this
Group=factom-walletd #This assumes group, if using root, delete this
EnvironmentFile=-/etc/default/factom-walletd
ExecStart=/usr/bin/factom-walletd $FACTOM_WALLETD_OPTS
KillMode=control-group
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
restart the wallet:
```
systemctl daemon-reload
systemctl start factom-walletd
```

Now factom-walletd should be running on localhost:8089, the default port. 
If you want to enable this to start on boot run
```
systemctl enable factom-walletd
```
Get a new factom address:
```
factom-cli newecaddress
```
***Get some testnet factoids:***
https://faucet.factoid.org/

Make sure your wallet's been funded:
```
factom-cli balance ECXXXXXXXXXXXXXX
```
Export your address:
```
factom-cli exportaddresses
```
Take note of your private TC address for the next step, beginning with "Esxxxxxxxxxxxxxxxx

*Download and build server identity program.*
The following are directions for linux.
Install git, golang 1.10, and glide. Set the proper $GOPATH environment variable.

```
sudo apt-get install git golang-go golang-glide
```

Edit ~/.profile and add these lines to the bottom (nano):
```
GOPATH=$HOME/go
PATH=$PATH:$GOPATH/bin
```
type:
```
source .profile
```

Clone the serveridentity program:
```
mkdir -p $GOPATH/src/github.com/FactomProject/
cd $GOPATH/src/github.com/FactomProject/
git clone https://github.com/FactomProject/serveridentity.git
cd serveridentity
glide install
go install
cd signwithed25519
go install
```

***The Tricky Part***
You'll need to use your wallet's private key as well as the serveridentity and signwithed22519
to create your server identity and generate the private keys for your server. This is where you'll need 
the offline computer and USB. 

Copy the serveridentity and signwithed25519 to offline computer

Run the following and entering in your private TC address from earlier:
```
serveridentity full elements Esxxxxxxxxxxxxxxxxxxxxx -n=important -f
```
***This will give you 4 secret keys, write them down***

This should also create two files important.conf and important.sh, they should be about 1kb and 4kb in size.
If not, rerun the above command.

Now, time to reverse the process.
Take important.conf and important.sh and use winscp to upload them to your factomd testnet server.
Edit the important.sh file (using nano) and add ```-s=localhost:8088``` right after ```factom-cli``` every time it appears.
Now run important.sh. ```sudo sh important.sh``` You should see a bunch of hashes, within 10 minutes the identifcation should be
written to the blockchain. 

You'll also need to modify your factomd.conf file with information from important.conf and restart factomd.

factomd.conf should be located here: /var/lib/docker/volumes/factom_keys/_data/factomd.conf

Take all information from important.conf, the file should look something like:
```
[app]
chainID 88888xxxxxxxx
............
...........
```

Copy all information (including [app]) to the end of the factomd.conf file replacing the default chainID, which looks something like this: ```fa1e0000000000000000```

Restart docker per the ***Updating Factomd*** code above 

You can check here to see your identity/chainID, just search for your public wallet address (ECxxxxxx....):

https://testnet.factoid.org/data?type=dblock-list

# Oh, you thought we were done?

We need to set our efficiency and coinbase address (coinbase refers to the Factoid address not the website). Before you can do this, you'll need to be admitted as a authority node, one of the Factom Admins can do this.

You'll need to be authorized onto the testnet or mainnet before this happens

First make sure you have node.js and npm installed, if not run:
```
curl -sL https://deb.nodesource.com/setup_11.x | sudo -E bash -
#after download is complete:
sudo apt-get install -y nodejs
```
Now install factom-identity-cli:
```
sudo npm install -g factom-identity-cli
```
Now for the next confusing part, you'll need your FCT address, your chainID, your management ID, and your level 1 secret from earlier (when you ran serveridentity)

if you don't have an FCT address/coins, just run ```factom-cli newfctaddress```, go to the faucet, and input the address

If you don't know the management ID, go to: http://136.144.191.225:8090 and search for your entry using your chainID, scroll down, the management ID is the second block with header Register Server Management starts with 88888

*Update coinbase address*

# Parameters: <Identity root chain ID> <FCT public address> <SK1 private key> <Paying private EC address>

```factom-identity-cli update-coinbase-address -s localhost:8088 .....```

*Update efficiency*

# Parameters: <Identity root chain ID> <Efficiency> <SK1 private key> <Paying private EC address>

 ```factom-identity-cli update-efficiency -s localhost:8088 ...```
 
 To see what your current efficieny and coinbase is:
 
 ```factom-identity-cli get -s localhost:8088 <ChainID>```
