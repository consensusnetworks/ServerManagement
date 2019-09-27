# Guide to attatching Servers to Prometheus and Grafana

This guide will walk you through how to use node exporter to attatch new servers to our existing prometheus/grafana
monitoring server. If you are installing a completely new monitoring server please see the documentation here: https://github.com/natemiller1/IoTeX_Monitoring

## Create Service Users
For security purposes, create two new user accounts, and use the --no-create-home and --shell /bin/false options so that these 
users can't log into the server.
```
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Create necessary directories:
```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

Change directory permissions:
```
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

## Downloading Node Exporter

Node Exporter is a prometheus package that scrapes the node for detailed information about the system including CPU, disk,
and memory usage. This data is then fed to prometheus and displayed via grafana. First, download the current stable version
of Node Exporter into your home directory. You can find the latest binaries along with their checksums on Prometheus' download 
page. ***NOTE***: v0.18.1 is the most recent version at this time, check the version before running otherwise you may not 
be able to install correctly.
```
cd ~
curl -LO https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
tar xvf node_exporter-0.18.1.linux-amd64.tar.gz
```
Copy the binary to the /usr/local/bin directory and set the user and group ownership to the node_exporter user that you created in Step 1.
```
sudo cp node_exporter-0.18.1.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Remove the leftover files from your directory.
```
rm -rf node_exporter-0.18.1.linux-amd64.tar.gz node_exporter-0.18.1.linux-amd64
```

## Running Node Exporter

Create a Systemd Service File for Node Exporter
```
sudo nano /etc/systemd/system/node_exporter.service
```
Then Copy the following contents into the file
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
Save the file & close your text editor. Now get systemd ready for the newly created service
```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl status node_exporter
sudo systemctl enable node_exporter
```
With Node Exporter fully configured and running as expected, we'll tell Prometheus to start scraping the new metrics.

Make sure you have the port open for prometheus on the servers and you should be good to go! Simply go the Grafana Dashboard, 
go to data sources and you should see options for the new server(s) you just added!


