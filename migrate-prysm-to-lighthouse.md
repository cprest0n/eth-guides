# How to switch from Prysm to Lighthouse

This guide will help those that want to migrate off the Prysm Beacon and Validator nodes that they have setup using the Somer Esat guides for [Staking on Ubuntu/Prysm](https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-prysm-56f681646f74). This guide assumes you have followed the guide exactly and your machine is currently running Geth and Prysm. This guide will follow (and copy from) the [Staking on Ubuntu/Lighthouse](https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-lighthouse-41de20513b12) guide. I was inspired by this [article](https://lighthouse.sigmaprime.io/switch-to-lighthouse.html) but wanted to make a detailed guide for myself and others to follow.

---

## Disclaimer

This guide is for informational purposes only and does not constitute professional advice. The author does not warrant or guarantee the accuracy, integrity, quality, completeness, currency, or validity of any information in this article. All information herein is provided “as is” without warranty of any kind and is subject to change at any time without notice. The author disclaims all express, implied, and statutory warranties of any kind, including warranties as to accuracy, timeliness, completeness, or fitness of the information in this article for any particular purpose. The author is not responsible for any direct, indirect, incidental, consequential or any other damages arising out of or in connection with the use of this article or in reliance on the information available on this article. This includes any personal injury, business interruption, loss of use, lost data, lost profits, or any other pecuniary loss, whether in an action of contract, negligence, or other misuse, even if the author has been informed of the possibility.

---

## Allow Lighthouse through the firewall

Part of step 4 in the Somer Esat guides

```
$ sudo ufw allow 9000
```

## Download Lighthouse

You can follow the entirety of step 7 of the [Somer Esat guide for Lighthouse](https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-lighthouse-41de20513b12) but I'll copy a trimmed down version here.

1. Go to the [Lighthouse Releases](https://github.com/sigp/lighthouse/releases)

2. Download the most recent version

```
$ cd ~
$ sudo apt install curl
$ curl -LO https://github.com/sigp/lighthouse/releases/download/v1.4.0/lighthouse-v1.4.0-x86_64-unknown-linux-gnu.tar.gz
```

3. Extract and copy to `/usr/local/bin`

```
$ tar xvf lighthouse-v1.4.0-x86_64-unknown-linux-gnu.tar.gz
$ sudo cp lighthouse /usr/local/bin
```

4. Verify the binary works with your CPU

```
$ cd /usr/local/bin/
$ ./lighthouse --version # <-- should display version information
```

> If this doesn't work, you will need to use the portable version of Lighthouse

5. Clean up

```
$ cd ~
$ sudo rm lighthouse
$ sudo rm lighthouse-v1.4.0-x86_64-unknown-linux-gnu.tar.gz
```

## Configure the Lighthouse Beacon node Service

This is step 9 of the [Somer Esat guide for Lighthouse](https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-lighthouse-41de20513b12). We are skipping step 8 of the guide until later. Like before, you should reference the guide but here is a trimmed down version.

1. Create a data directory for lighthouse

```
$ sudo mkdir -p /var/lib/lighthouse
$ sudo chown -R <yourusername>:<yourusername> /var/lib/lighthouse
```

> Changing the ownership of the lighthouse directory is necessary for a later step

2. Set up the Beacon Node Account and Directory

```
$ sudo useradd --no-create-home --shell /bin/false lighthousebeacon
$ sudo mkdir -p /var/lib/lighthouse/beacon
$ sudo chown -R lighthousebeacon:lighthousebeacon /var/lib/lighthouse/beacon
$ sudo chmod 700 /var/lib/lighthouse/beacon
```

3. Create the Lighthouse Beacon node systemd service file

```
$ sudo nano /etc/systemd/system/lighthousebeacon.service
```

Then paste the following:

```
[Unit]
Description=Lighthouse Eth2 Client Beacon Node
Wants=network-online.target
After=network-online.target

[Service]
User=lighthousebeacon
Group=lighthousebeacon
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse bn --network mainnet --datadir /var/lib/lighthouse --staking --eth1-endpoints http://127.0.0.1:8545

[Install]
WantedBy=multi-user.target
```

> Check the guide or the official documentation for additional flags/settings

4. Start the Lighthouse Beacon node and enable the service to automatically start

Running two beacon nodes at the same time is fine. This will not cause a slashing event.

```
$ sudo systemctl daemon-reload
$ sudo systemctl start lighthousebeacon
$ sudo systemctl status lighthousebeacon
$ sudo systemctl enable lighthousebeacon
```

The beacon node should now be syncing and you can follow the progress here

```
$ sudo journalctl -fu lighthousebeacon.service
```

## Configure the Lighthouse Validator Service

> **WARNING: DO NOT START THIS LIGHTHOUSE VALIDATOR SERVICE WHILE YOUR PRYSM VALIDATOR SERVICE IS RUNNING. THIS WILL EVENTUALLY CAUSE A SLASHING EVENT**

This section uses parts of step 8 and 10 of the [Somer Esat guide for Lighthouse](https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-lighthouse-41de20513b12). Like before, you should reference the guide but here is a trimmed down version.

1. Run the validator key import process (make sure the directory still exists and has your keys in them)

```
$ cd /usr/local/bin
$ lighthouse --network mainnet account validator import --directory $HOME/eth2deposit-cli/validator_keys --datadir /var/lib/lighthouse
```

Provide the password you used when you created the validator keys initially.

> If for some reason your keys are no longer on your machine, regenerate them by following either of the guides or other official documentation.

2. Set default permissions to the parent lighthouse directory

```
$ sudo chown root:root /var/lib/lighthouse
```

3. Set up the Validator Node Account and Directory

```
$ sudo useradd --no-create-home --shell /bin/false lighthousevalidator
```

4. Set permissions for the wallet data

```
$ sudo chown -R lighthousevalidator:lighthousevalidator /var/lib/lighthouse/validators
$ sudo chmod 700 /var/lib/lighthouse/validators
```

5. Create the Lighthouse Validator service systemd service file

```
$ sudo nano /etc/systemd/system/lighthousevalidator.service
```

Then paste the following: 

```
[Unit]
Description=Lighthouse Eth2 Client Validator Node
Wants=network-online.target
After=network-online.target

[Service]
User=lighthousevalidator
Group=lighthousevalidator
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse vc --network mainnet --datadir /var/lib/lighthouse --graffiti "<yourgraffiti>"

[Install]
WantedBy=multi-user.target
```

> **WARNING: AGAIN DO NOT START THIS SERVICE WHILE YOUR PRYSM SERVICE IS STILL RUNNING**

## Switching from Prysm to Lighthouse

Here we will finally switch our validator services to Lighthouse

> **WARNING: Follow these next steps carefully to not cause a slashing event**

1. Wait for / make sure the Lighthouse Beacon node has fully synced and has no errors

```
$ sudo journalctl -fu lighthousebeacon.service
```

2. Stop and disable the Prysm validator client

```
$ sudo systemctl stop prysmvalidator.service
$ sudo systemctl disable prysmvalidator.service
```

You can also optionally stop and disable the Prysm Beacon node

```
$ sudo systemctl stop prysmbeacon.service
$ sudo systemctl disable prysmbeacon.service
```

3. (Optional) Disable your public keys in the Prysm Validator to make sure that even if it is started, it won't cause a slashing event 

```
$ cd /usr/local/bin
$ validator accounts --disable-public-keys <comma-separated list of public key hex strings>
```

4. Export Prysm validator slashing protection history

```
$ mkdir $HOME/slashprotection
$ cd /usr/local/bin
$ validator slashing-protection export --datadir=/var/lib/prysm/validator --slashing-protection-export-dir=$HOME/slashprotection
```

5. Check that your keys are in Lighthouse

```
$ sudo lighthouse account validator list --datadir=/var/lib/lighthouse
```

6. Import the slashing protection history into Lighthouse

```
$ sudo lighthouse account validator slashing-protection import $HOME/slashprotection/slashing_protection.json --datadir=/var/lib/lighthouse
```

7. Start the Lighthouse Validator node and enable the service to automatically start

```
$ sudo systemctl daemon-reload
$ sudo systemctl start lighthousevalidator
$ sudo systemctl status lighthousevalidator
$ sudo systemctl enable lighthousevalidator
$ sudo journalctl -fu lighthousevalidator.service
```

### Done! You should now be using the Lighthouse Beacon node and Validator node.

---

## Appendix A - Removing Prysm

If you do not plan on using Prysm at all, follow these steps to remove it from your machine.

1. If you didn't stop and disable the Prysm Beacon node already

```
$ sudo systemctl stop prysmbeacon.service
$ sudo systemctl disable prysmbeacon.service
```

2. Remove the binaries

```
$ cd /usr/local/bin
$ sudo rm validator beaconchain
```

3. Remove the prysm directory

```
$ sudo rm -r /var/lib/prysm
```

4. Remove the systemd service files

```
$ sudo rm /etc/systemd/system/prysmbeacon.service
$ sudo rm /etc/systemd/system/prysmvalidator.service
$ sudo systemctl daemon-reload
```

5. Remove firewall rules for Prysm

```
$ sudo ufw delete allow 13000/tcp
$ sudo ufw delete allow 12000/udp
```
