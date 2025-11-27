<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Staking Ethereum with Lighthouse \ Ubuntu - Mainnet](#staking-ethereum-with-lighthouse-%5C-ubuntu---mainnet)
  - [Install Prerequisites](#install-prerequisites)
  - [Update the Server](#update-the-server)
  - [Secure the Server](#secure-the-server)
  - [Configure Timekeeping](#configure-timekeeping)
  - [Set up an Ethereum (Eth1) Node](#set-up-an-ethereum-eth1-node)
  - [Download Lighthouse](#download-lighthouse)
  - [Import the Validator Keys](#import-the-validator-keys)
- [Configure the Beacon Node Service](#configure-the-beacon-node-service)
  - [Create and Configure the Service](#create-and-configure-the-service)
- [Configure the Validator Service](#configure-the-validator-service)
  - [Set up the Validator Node Account and Directory](#set-up-the-validator-node-account-and-directory)
  - [Create and Configure the Service](#create-and-configure-the-service-1)
- [Updating Geth](#updating-geth)
- [Updating Lighthouse](#updating-lighthouse)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

Staking Ethereum with Lighthouse \ Ubuntu - Mainnet
===================================================

Install Prerequisites
---------------------

Install Codecopy

https://github.com/zenorocha/codecopy#install


Update the Server
-----------------

Make sure the system is up to date with the latest software and security updates.

```
sudo apt update && sudo apt upgrade -y
```
```
sudo apt dist-upgrade && sudo apt autoremove
```
```
sudo reboot
```

Secure the Server
-----------------

Find your available port.

```
sudo ss -tulpn | grep ':<yourSSHportnumber>'
```
  
Update the firewall to allow inbound traffic on <yourSSHportnumber>. SSH requires TCP.

```
sudo ufw allow <yourSSHportnumber>/tcp
```
  
Next change the default SSH port.

```
sudo nano /etc/ssh/sshd_config
```

Find the line with # Port 22 or Port 22 and change it to Port <yourSSHportnumber>. Remove the # if it was present.
Restart the SSH service.
  
```  
sudo systemctl restart ssh
```

Next time you log in via SSH use <yourSSHportnumber> for the port.
Optional: If you were already using UFW with port 22/TCP allowed then update the firewall to deny inbound traffic on that port. Only do this after you log in using the new SSH port.

```
sudo ufw deny 22/tcp
```

Install UFW
UFW should be installed by default. The following command will ensure it is.

```
sudo apt install ufw
```

Apply UFW Defaults
Explicitly apply the defaults. Inbound traffic denied, outbound traffic allowed.

```
sudo ufw default deny incoming
```
```
sudo ufw default allow outgoing
```

Create and run this script or download via wget.

```
#!/bin/bash
#Allow Go Ethereum
sudo ufw allow 30303
#Allow Lighthouse
sudo ufw allow 9000
#Allow Grafana
sudo ufw allow 3000/tcp
#Allow Prometheus
sudo ufw allow 9090/tcp
#Enable Firewall
sudo ufw enable
sudo ufw status numbered
```
Note: Geth node Port 8545wget firewall.sh

```
wget https://raw.githubusercontent.com/ipalvr/ethstaking_prysm_pyrmont/main/firewall.sh
```

Configure Timekeeping
---------------------

Ubuntu has time synchronization built in and activated by default using systemd’s timesyncd service. Verify it’s running correctly.

```
timedatectl
```

The NTP service should be active. If not then run:

```
sudo timedatectl set-ntp on
```

You should only be using a single keeping service. If you were using NTPD from a previous installation you can check if it exists and remove it using the following commands.

```
ntpq -p
```
```
sudo apt-get remove ntp
```

Set up an Ethereum (Eth1) Node
------------------------------

An Ethereum node is required for staking. You can either run a local Eth1 node or use a third party node. This guide will provide instructions for running Go Ethereum. If you would rather use a third party option then skip this step.

Install Go Ethereum
Install the Go Ethereum client using PPA’s (Personal Package Archives).

```
sudo add-apt-repository -y ppa:ethereum/ethereum
```
```
sudo apt update
```
```
sudo apt install geth
```

Go Ethereum will be configured to run as a background service. Create an account for the service to run under. This type of account can’t log into the server.

```
sudo useradd --no-create-home --shell /bin/false goeth
```

Create the data directory for the Eth1 chain. This is required for storing the Eth1 node data.

```
sudo mkdir -p /var/lib/goethereum
```

Set directory permissions. The goeth account needs permission to modify the data directory.

```
sudo chown -R goeth:goeth /var/lib/goethereum
```

Create a systemd service config file to configure the service.

```
sudo nano /etc/systemd/system/geth.service
```

Paste the following service configuration into the file and save or download via wget from the link below.

```
[Unit]
Description=Go Ethereum Client
After=network.target
Wants=network.target
[Service]
User=goeth
Group=goeth
Type=simple
Restart=always
RestartSec=5
ExecStart=geth --http --datadir /var/lib/goethereum --cache 2048 --maxpeers 30
[Install]
WantedBy=default.target
```
Note:  Make sure you change to the /etc/systemd/system directory.

wget https://raw.githubusercontent.com/ipalvr/ethstaking_prysm_pyrmont/main/geth.service

Notable flags:
--http Expose an HTTP endpoint (http://localhost:8545) that the Lighthouse beacon chain will connect to.
--cache Size of the internal cache in GB. Reduce or increase depending on your available system memory. A setting of 2048 results in roughly 4–5GB of memory usage.
--maxpeers Maximum number of peers to connect with. More peers equals more internet data usage. Do not set this too low or your Eth1 node will struggle to stay in sync.

Reload systemd to reflect the changes and start the service. Check status to make sure it’s running correctly.

```
sudo systemctl daemon-reload
```

```
sudo systemctl start geth
```

```
sudo systemctl status geth
```

It should say active (running) in green text. If not then go back and repeat the steps to fix the problem. Press Q to quit (will not affect the geth service).

Enable the geth service to automatically start on reboot.

```
sudo systemctl enable geth
```

The Go Ethereum node will begin to sync. You can follow the progress or check for errors by running the following command. Press Ctrl+C to exit (will not affect the geth service).

```
sudo journalctl -fu geth.service
```

Check Sync Status - To check your Eth1 node sync status use the following command to access the console.

```
geth attach http://127.0.0.1:8545
```
Type...
```  
eth.syncing
```
If false is returned then your sync is complete. If syncing data is returned then you are still syncing. For reference there are roughly 700–800 million knownStates.

Here is another way to verify if geth is syncing.

```
curl --request POST localhost:8545 \
    --header 'Content-type: application/json' \
    --data-raw '{
    "jsonrpc":"2.0",
    "method":"eth_syncing",
    "params":[],
    "id":1
    }'
```
If false is returned then your sync is complete.
  
Download Lighthouse
-------------------
The Lighthouse client is a single binary which encapsulates the functionality of the beacon chain and validator. This step will download and prepare the Lighthouse binary.
First, go to the link below and identify the latest release. It is at the top of the page. For example:

https://github.com/sigp/lighthouse/releases

Download the archive using the commands below. Modify the URL in the instructions below to match the download link for the latest version.

```
cd ~
```
```
sudo apt install curl
```
```
curl -LO https://github.com/sigp/lighthouse/releases/download/v1.0.2/lighthouse-v1.0.2-x86_64-unknown-linux-gnu.tar.gz
```

Extract the binary from the archive and copy to the /usr/local/bin directory. The Lighthouse service will run it from there. Modify the URL name as necessary.

```
tar xvf lighthouse-v1.0.2-x86_64-unknown-linux-gnu.tar.gz
```
```
sudo cp lighthouse /usr/local/bin
```

Use the following commands to verify the binary works with your server CPU. If not, go back and download the portable version and redo the steps to here and try again.

```
cd /usr/local/bin/
```
```
./lighthouse --version # <-- should display version information
```

NOTE: There has been at least one case where version information is displayed yet subsequent commands have failed. If you get a Illegal instruction (core dumped) error while running the account validator import command (next step), then you may need to use the portable version instead.
Clean up the extracted files.

```
cd ~
```
```
sudo rm lighthouse
```
```
sudo rm lighthouse-v1.0.2-x86_64-unknown-linux-gnu.tar.gz
```

NOTE: It is necessary to follow a specific series of steps to update Lighthouse. See Appendix B — Updating Lighthouse for further information.

Import the Validator Keys
-------------------------

Configure Lighthouse by importing the validator keys and creating the service and service configuration required to run it.

Copy the Validator Keystore Files

If you generated the validator keystore-m…json file(s) on a machine other than your Ubuntu server you will need to copy the file(s) over to your home directory. You can do this using a USB drive (if your server is local), or via secure FTP (SFTP).

Place the files here: $HOME/eth2deposit-cli/validator_keys. Create the directories if necessary.

Import Keystore Files into the Validator Wallet

Create a directory to store the validator wallet data and give the current user permission to access it. The current user needs access because they will be performing the import. Change <yourusername> to the logged in username.

```
sudo mkdir -p /var/lib/lighthouse
```
```
sudo chown -R <yourusername>:<yourusername> /var/lib/lighthouse
```

Copy keys via scp.
```
scp -P 8675 keystore-m_xxxxxxxxx.json username@192.168.1.2:eth2deposit-cli/validator_keys
```

Run the validator key import process. You will need to provide the directory where the generated keystore-m files are located. E.g. $HOME/eth2deposit-cli/validator_keys.

```
cd /usr/local/bin
```
```
lighthouse --network mainnet account validator import --directory $HOME/eth2deposit-cli/validator_keys --datadir /var/lib/lighthouse
```

You will be asked to provide the password for the validator keys. This is the password you set when you created the keys during Step 1.

You will be asked to provide the password for each key, one-by-one. Be sure to correctly provide the password each time because the validator will be running as a service and it needs to persist the password(s) to a file to access the key(s).

Note that the validator data is saved in the following location created during the keystore import process: /var/lib/lighthouse/validators.

Restore default permissions to the lighthouse directory.

```
sudo chown -R root:root /var/lib/lighthouse
```

Configure the Beacon Node Service
=================================


In this step you will configure and run the Lighthouse beacon node as a service so if the system restarts the process will automatically start back up again.

Set up the Beacon Node Account and Directory

Create an account for the beacon node to run under. This type of account can’t log into the server.

```
sudo useradd --no-create-home --shell /bin/false lighthousebeacon
```
Create the data directory for the Lighthouse beacon node database and set permissions.
```
sudo mkdir -p /var/lib/lighthouse/beacon
```
```
sudo chown -R lighthousebeacon:lighthousebeacon /var/lib/lighthouse/beacon
```
```
sudo chmod 700 /var/lib/lighthouse/beacon
```
```
ls -dl /var/lib/lighthouse/beacon
```

Create and Configure the Service
--------------------------------

Create a systemd service config file to configure the service.

```
sudo nano /etc/systemd/system/lighthousebeacon.service
```
Paste the following into the file.
```
[Unit]
Description=Lighthouse Consensus Client BN (Mainnet)
Wants=network-online.target
After=network-online.target
[Service]
User=lighthousebeacon
Group=lighthousebeacon
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse bn \
  --network mainnet \
  --datadir /var/lib/lighthouse \
  --http \
  --execution-endpoint http://127.0.0.1:8551 \
  --execution-jwt /var/lib/jwtsecret/jwt.hex \
  --checkpoint-sync-url https://2EDVFgy7GeRoyge8NqS0OwRcOjK:5ace00240f62666218e9a5377b1a0e2e@eth2-beacon-mainnet.infura.io \
  --monitoring-endpoint https://beaconcha.in/api/v1/client/metrics?apikey=SzN3dFRheVR0ekN6ZmpXMGVOYmll&machine=asuspn50
[Install]
WantedBy=multi-user.target
```

Notable flags
bn subcommand instructs the lighthouse binary to run a beacon node.
--eth1-endpoints One or more comma-delimited server endpoints for web3 connection. If multiple endpoints are given the endpoints are used as fallback in the given order. Also enables the -- eth1 flag. E.g. --eth1-endpoints http://127.0.0.1:8545,https://yourinfuranode,https://your3rdpartynode.

Reload systemd to reflect the changes and start the service.

```
sudo systemctl daemon-reload
```
Note: If you are running a local Eth1 node (see Step 6) you should wait until it fully syncs before starting the lighthousebeacon service. Check progress here: sudo journalctl -fu geth.service
Start the service and check to make sure it’s running correctly.
```
sudo systemctl start lighthousebeacon
```
```
sudo systemctl status lighthousebeacon
```

Enable the service to automatically start on reboot.
```
sudo systemctl enable lighthousebeacon
```
If the Eth2 chain is post-genesis the Lighthouse beacon chain will begin to sync. It may take several hours to fully sync. You can follow the progress or check for errors by running the journalctl command. Press CTRL+C to exit (will not affect the lighthousebeacon service).
```
sudo journalctl -fu lighthousebeacon.service
```
A truncated view of the log shows the following status information.

[NOTE: A current issue is resulting in an incorrect error message.]

INFO Waiting for genesis 

wait_time: 5 days 5 hrs, peers: 50, service: slot_notifier

Once the Eth2 mainnet starts up the beacon chain will automatically start processing. The output will give an indication of time to fully sync with the Eth1 node.

Configure the Validator Service
===============================

In this step you will configure and run the Lighthouse validator node as a service so if the system restarts the process will automatically start back up again.

Set up the Validator Node Account and Directory
-----------------------------------------------

Create an account for the validator node to run under. This type of account can’t log into the server.
```
sudo useradd --no-create-home --shell /bin/false lighthousevalidator
```
In the validator wallet creation process we created the following directory: /var/lib/lighthouse/validators. Set directory permissions so the lighthousevalidator account can modify that directory.
```
sudo chown -R lighthousevalidator:lighthousevalidator /var/lib/lighthouse/validators
```
```
sudo chmod 700 /var/lib/lighthouse/validators
```
```
ls -dl /var/lib/lighthouse/validators
```

Create and Configure the Service
--------------------------------

Create a systemd service file to store the service config.
```
sudo vim /etc/systemd/system/lighthousevalidator.service
```

Paste the following into the file.

```
Unit
Description=Lighthouse Consensus Client VC (Mainnet)
Wants=network-online.target
After=network-online.target
[Service]
User=lighthousevalidator
Group=lighthousevalidator
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse vc \
  --network mainnet \
  --datadir /var/lib/lighthouse \
  --enable-doppelganger-protection \
  --suggested-fee-recipient 0xb30b1311112B535c5E8aE86Ef8dD1B7f76017CD2 \
  --graffiti "Hello from ipalvr!" \
  --monitoring-endpoint https://beaconcha.in/api/v1/client/metrics?apikey=SzN3dFRheVR0ekN6ZmpXMGVOYmll&machine=asuspn50
[Install]
WantedBy=multi-user.target

```

Notable flags.
BindsTo=lighthousebeacon.service will stop the validator service if the beacon service stops. The validator service cannot function without the beacon service.

vc subcommand instructs the lighthouse binary to run a validator node.

--graffiti "<yourgraffiti>" Replace with your own graffiti string. For security and privacy reasons avoid information that can uniquely identify you. E.g. --graffiti "Hello Eth2! From Dominator".

Reload systemd to reflect the changes and start the service and check to make sure it’s running correctly.
```
sudo systemctl daemon-reload
```
```
sudo systemctl start lighthousevalidator
```
```
sudo systemctl status lighthousevalidator
```

Enable the service to automatically start on reboot.
```
sudo systemctl enable lighthousevalidator
```

You can follow the progress or check for errors by running the journalctl command. Press CTRL+C to exit (will not affect the lighthousevalidator service.)
```
sudo journalctl -fu lighthousevalidator.service
```

For post-genesis deposits it may take hours or even days to activate the validator account(s) once the beacon chain has started processing.
Once the Eth2 mainnet starts up the beacon chain and validator will automatically start processing.

Once the Eth2 mainnet starts up the beacon chain and validator will automatically start processing.

Upgrading Besu
==============

Get the lastest version of Besu.

```  
/usr/local/bin/besu/bin/besu --version  
```
  
Click here for the latest release of Besu. 

https://github.com/hyperledger/besu/releases

In the Download link section copy the download link to the tar.gz file. Be sure to copy the correct link.  Modify the URL in the instructions below to match the download link for the latest version.

```
cd ~
```

```
curl -LO https://hyperledger.jfrog.io/artifactory/besu-binaries/besu/xx.x.x/besu-xx.x.x.tar.gz
```
Verify the file.

```
sha256sum --check besu-25.11.0.tar.gz.sha256
```

Stop the Besu service.

```
sudo systemctl stop besu
```

Extract the files from the archive and copy to the /usr/local/bin directory. Modify the file name to match the downloaded version.

```
tar xvf besu-xx.x.x.tar.gz
```

Remove the old files

```
sudo rm -r /usr/local/bin/besu
```
```
sudo cp -a besu-xx.x.x /usr/local/bin/besu
```

Restart the services and check for errors.

```
sudo systemctl start besu
```

Check for errors

```
sudo systemctl status besu
```

Monitor

```
sudo journalctl -fu besu
```

```
sudo journalctl -fu lighthousebeacon
```

Clean up the files. Modify the file name to match the downloaded version.

```
cd ~
```

```
rm besu-xx.x.x.tar.gz
```

```
rm -r besu-xx.x.x
```

Updating Lighthouse
===================

If you need to update to the latest version of Lighthouse follow these steps.

First, go here and identify the latest Linux release. Modify the URL in the instructions below to match the download link for the latest version.

NOTE: There are two types of binaries — portable and non-portable.  The -portable suffix which indicates if the portable feature is used:
Without portable: uses modern CPU instructions to provide the fastest signature verification times (may cause Illegal instruction error on older CPUs)
With portable: approx. 20% slower, but should work on all modern 64-bit processors.  More info here:
https://lighthouse-book.sigmaprime.io/installation-binaries.html
  
Link to latest bininaries:
https://github.com/sigp/lighthouse/releases

Download PGP Signature
```
curl -LO https://github.com/sigp/lighthouse/releases/download/VERSION/lighthouse-VERSION-ARCHITECTURE-unknown-linux-gnu.tar.gz.asc
```
Download Binary
```
curl -LO https://github.com/sigp/lighthouse/releases/download/VERSION/lighthouse-VERSION-ARCHITECTURE-unknown-linux-gnu.tar.gz
```
Verify PGP Key
```
gpg --verify lighthouse-VERSION-ARCHITECTURE-unknown-linux-gnu.tar.gz.asc
```
Stop the Lighthouse client services.
```
sudo systemctl stop lighthousevalidator
```
```
sudo systemctl stop lighthousebeacon
```
Extract the binary from the archive and copy to the /usr/local/bin directory. Modify the URL name as necessary.
```
tar xvf lighthouse-VERSION-ARCHITECTURE-unknown-linux-gnu.tar.gz
```
```
sudo cp lighthouse /usr/local/bin
```
Check version
```
/usr/local/bin/lighthouse -V
```
Restart the Beacon service and check for errors
```
sudo systemctl start lighthousebeacon
```
Check for errors
```
sudo systemctl status lighthousebeacon
```
Monitor
```
sudo journalctl -fu lighthousebeacon
```
Restart the Validator service and check for errors
```
sudo systemctl start lighthousevalidator
```
Check for errors
```
sudo systemctl status lighthousevalidator
```
Monitor  
```
sudo journalctl -fu lighthousevalidator
```
Clean up the extracted files.
```
cd ~
```
```
sudo rm lighthouse
```
```
sudo rm lighthouse-VERSION-ARCHITECTURE-unknown-linux-gnu.tar.gz
```

