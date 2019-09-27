# Factom Testnet server setup, configuration, and operating instructions

## Prereqs and Config

*Supplementa lLinks*
http://www.factom-testnet.com/Introduction

### The server needs to be configured to listen to the following ports for the following IP addresses 

TCP port 2376 open to 52.48.130.243/32 & 18.203.51.247/32 for secure Docker engine communication

2222 to 52.48.130.243/32 & 18.203.51.247/32, which is the SSH port used by the ssh container

8088 to 52.48.130.243/32 & 18.203.51.247/32, the factomd API port

8090 to 52.48.130.243/32 & 18.203.51.247/32, the factomd Control panel, Keeping this open to the world is beneficial on testnet for debugging purposes

8108 to 0.0.0.0, the factomd mainnet port

On Ubuntu this is done as follows:
```
sudo ufw enable
sudo ufw allow from 18.203.51.247 to any port 2376
sudo ufw allow from 18.203.51.247 to any port 2222
sudo ufw allow from 18.203.51.247 to any port 8088
sudo ufw allow from 0.0.0.0/0 to any port 8090
sudo ufw allow from 0.0.0.0/0 to any port 8108
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
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/cert_exp_5-14-21.pem -O /etc/docker/factom-mainnet-cert_exp_5-14-21.pem
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/key_exp_5-14-21.pem -O /etc/docker/factom-mainnet-key_exp_5-14-21.pem
 sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/ca_exp_5-14-21.pem -O /etc/docker/factom-mainnet-ca_exp_5-14-21.pem
sudo chmod 644 /etc/docker/factom-mainnet-cert_exp_5-14-21.pem
sudo chmod 440 /etc/docker/factom-mainnet-key_exp_5-14-21.pem /etc/docker/factom-mainnet-ca_exp_5-14-21.pem
sudo chgrp docker /etc/docker/*.pem
```

Create a json file, this will help start factomd through docker:

```sudo nano /etc/docker/daemon.json```
Paste this in the file (ensure path is correct!)
```
{
  "tlsverify": true,
  "tlscert": "/etc/docker/factom-mainnet-cert_exp_5-14-21.pem",
  "tlskey": "/etc/docker/factom-mainnet-key_exp_5-14-21.pem",
  "tlscacert":"/etc/docker/factom-mainnet-ca_exp_5-14-21.pem",
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
## Starting the Node

Create the FactomD volumes to hold your identity keys and the database 
```
docker volume create factom_database
docker volume create factom_keys
```
### Setting the config file

Need to create factom.conf file, this will initialize the factom docker instance when it starts:

Add a .conf file using the following command and paste the contents from this file inside https://raw.githubusercontent.com/FactomProject/factomd/master/factomd.conf
```
sudo nano /var/lib/docker/volumes/factom_keys/_data/factomd.conf
```

Now you must add some special peers. Inside the conf file, under the [app] line, look for line starting with MainSpecialPeers and the following peers so that the line looks as below:
```
MainSpecialPeers     = "52.17.183.121:8108 52.17.153.126:8108 52.19.117.149:8108 52.18.72.212:8108"
```

### Join the Factom  Swarm:
```
docker swarm join --token SWMTKN-1-5ct5plmbn1ombbjqp8ql8hq93jkof6246suzast5n1gfwa083b-1ui6w6fupe45tizz0tv6syzrs 52.48.130.243:2377
```

Run Factomd:
*** Pay attention to version *** (v6.3.3-alpine as of writing this)
```
docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8108:8108" -l "name=factomd" factominc/factomd:v6.3.2-alpine -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
```
Now let the node sync.....

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

## Updating Factomd/Brainswaps (need to pay attention to ***version***):
```
sudo docker stop factomd
sudo docker rm factomd
sudo docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8110:8110" -l "name=factomd" factominc/factomd:v6.2.3-rc3-alpine -broadcastnum=16 -network=CUSTOM -customnet=fct_community_test -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
```
Brainswapping is the idea of having 2 nodes switch identities at the same time. We call is "brainswapping" because a node's identity dictates how it behaves.

Note: The procedure doesn't actually have to be a "swap"; a "brain-transfer" is also an alternative.

It was implemented as a way to update the network without having to bring it down, as federated servers can "brainswap" with standby nodes that have already been updated with the new code.

The network will not perceive this as a node going offline, as the identity (and thus the associated federated server) is still online.

After transferring the Authority identity the node operator can now shutdown their original node, perform necessary updates, bring it back online and finally brainswap the identity back into the original server.

The procedure can also be used for migrating the Authority identity to a new physical server, by not performing the brainswap a second time to reverse the first swap.

Definitions
Federated Node is the node that holds your authority identity, and needs to be updated.
Standby Node is a follower node on the network that you control.
Prepping the Brain Swap
To perform the brain swap you will need 1 standby node that is ready. The standby node should be running the most recent Factomd software version.

Determining if you Standby Node is ready
If your standby node is not in sync with the network, performing the brainswap will result in your authority server going offline, so it is crucial to first check the health of the standby node.

Check if the DBHeight matches that of the network
Check if the minutes are following that of the network
Check the process list for <nil>, that indicates some network instability (Procedures for the above described at the bottom of this document)
Performing the Brain Swap (Read through before performing)
Once your standby node is ready, we can prep for the swap. The swap requires both config files (located on Federated Node and Standby Node) to be modified.

Open both config files in parallel in a text editor of your choice. (Procedures for editing is described at the bottom of this document)

Swap the following lines in the two config files:

IdentityChainID	                      = FA1E000000000000000000000000000000000000000000000000000000000000
LocalServerPrivKey                = 4c38c72fc5cdad68f13b74674d3ffb1f3d63a112710868c9b08946553448d26d
LocalServerPublicKey            = cc1985cdfae4e32b5a454dfda8ce5e1361558482684f3367649c3ad852c8e31a
(If you prefer to do a brain-transfer instead of a swap, you can just comment out the lines in your Federated node by placing a ; in front of these lines.)

Then add an additional line:

ChangeAcksHeight                      = 0
This additional line is the brainswap logic. You will want to set the ChangeAcksHeight to some block height in the future (remember the block height on the control panel is the last saved height, not the current working height!).

The safe height is the one you see in the control panel + 4. (localhost:8090). If you know how to read the more detailed page, you can get away with a closer number.

Once you set the ChangeAcksHeight to DBHeight+4, save both files.

At the block height ChangeAcksHeight you should see both nodes change identities. If none of your nodes crash, and the identities change, the swap was successful.


