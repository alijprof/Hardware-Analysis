# Hardware-Analysis
Hardware analysis of Celestia Node 

## Introduction

The following article will explain how you can analyse the hardware of your VPS or local machine for the celestia ITN tasks **Perform analysis of your node.** It will also cover any interesting analytics that has been picked up from monitoring my VPS which is running both a Bridge and Consensus Node. 

The analysis was done by hosting a separate VPS that runs prometheus and node exporter and linking it to my VPS running my Celestia Nodes. This could also have been run on the same VPS that my Celestia Nodes were running on but to get a more accurate analysis of just the nodes performance a dedicated server was set up to scrape the ITN VPS hardware analysis. 

### Analytical tools used

As mentioned above 2 VPS were rented and the software on each are described below:

| Dedicated Prometheus VPS | Celestia ITN VPS |
| --- | --- |
| Prometheus  | Celestia-App |
| Node Exporter | Celestia-Node |
| Grafana | Node Exporter |
| -  | Go |

Grafana is a popular open-source data visualisation tool used to create custom dashboards that display real-time metrics from various data sources. Prometheus is a monitoring system and time-series database that collects and stores metrics from targets such as servers, applications, and services. Node Exporter is a Prometheus exporter that collects and exposes system-level metrics from Linux and Unix systems.

By using Prometheus and Node Exporter together, we can scrape information from our Celestia ITN VPS and store it in Prometheus's time-series database. We can then use Grafana to visualise these metrics in real-time and create custom dashboards that provide insight into our Celestia ITN VPS hardware performance.

## Openning Ports

The following ports should be run on the Prometheus VPS:

```jsx
sudo ufw allow ssh 
sudo ufw allow 9090
sudo ufw allow 9100
sudo ufw allow 3000
sudo ufw enable
```

The following ports should be run on the Celestia ITN VPS:

```jsx
sudo ufw allow ssh 
sudo ufw allow 9100
sudo ufw enable
```

# **Running Prometheus and Grafana on a dedicated server**

For the following work as root user on your dedicated prometheus server. Command `sudo -i`

1. Download Prometheus and Extract (Promtol and Prometheus)

```jsx
wget [https://github.com/prometheus/prometheus/releases/download/v2.33.0/prometheus-2.33.0.linux-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.33.0/prometheus-2.33.0.linux-amd64.tar.gz)
```

```jsx
tar xvf prometheus-2.33.0.linux-amd64.tar.gz
```

1. Move the Prometheus binaries to /usr/local/bin

```jsx
sudo cp prometheus-2.33.0.linux-amd64/prometheus /usr/local/bin/
```

```jsx
sudo cp prometheus-2.33.0.linux-amd64/promtool /usr/local/bin/
```

1. grant permissions (assumes user is ubuntu)

```jsx
sudo chown ubuntu:ubuntu /usr/local/bin/prometheus
```

```jsx
sudo chown ubuntu:ubuntu /usr/local/bin/promtool
```

1. Check installed prometheus version:

```jsx
prometheus --version
```

*Example output from above command:*

![Screenshot 2023-05-06 at 15.11.21.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0fdd8f96-9895-4e39-8d19-a58c460713e1/Screenshot_2023-05-06_at_15.11.21.png)

1. Check installed promtool version:

```jsx
promtool --version
```

*Example output from above command*

![Screenshot 2023-05-06 at 15.11.51.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c06cdbe4-f972-4a32-b034-8ce9d12292bd/Screenshot_2023-05-06_at_15.11.51.png)

1. Download and Extract Node exporter 

The following 6, 7 and 8 steps should be done on both the prometheus VPS and the Celestia ITN VPS.

```jsx
wget [https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz)
```

```jsx
tar xvf node_exporter-1.3.1.linux-amd64.tar.gz
```

1. Move and Grant Permissions

```jsx
sudo cp node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/bin/
```

```jsx
sudo chown ubuntu:ubuntu /usr/local/bin/node_exporter
```

1. Check installed node exporter version:

```jsx
node_exporter --version
```

*Example output from above command:*

![Screenshot 2023-05-06 at 15.15.54.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9625ddf4-38ea-464e-ac41-4a7b8f134659/Screenshot_2023-05-06_at_15.15.54.png)

## Configure and run Prometheus and Node Exporter and SystemD Service.

SystemD is a daemon service useful for running applications as background processes.

1. Create Prometheus configuration file:

```jsx
sudo mkdir /etc/prometheus/
```

**IMPORTANT - In the following code make sure to replace <YOUR-IP> with the IP address of your node VPS you want to scrape. If running locally, change to localhost.** 

```
sudo tee /etc/prometheus/prometheus.yml << EOF
global:
  scrape_interval: 10s
  scrape_timeout: 3s
  evaluation_interval: 5s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node_exporter
    static_configs:
      - targets: ['<YOUR-IP>:9100']
EOF
```

1. Create a system service for Prometheus

```jsx
sudo tee /etc/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=root
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF
```

1. Create a system service for Node Exporter

```jsx
sudo tee /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=root
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

1. Enable and Start Services

```jsx
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

1. Check prometheus and node_exporter are running:

```jsx
sudo systemctl status prometheus
```

*Example output from above command:*

![Screenshot 2023-05-07 at 23.13.04.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4d16f48b-fd1c-42e3-b20f-5b8f199d0060/Screenshot_2023-05-07_at_23.13.04.png)

```jsx
sudo systemctl status node_exporter
```

*Example output from above command:*

![Screenshot 2023-05-07 at 23.13.40.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/617b4517-a983-4d78-9ebb-f9af00bcc387/Screenshot_2023-05-07_at_23.13.40.png)

## Installing Grafana

**Installing Grafana on the Prometheus server (can also use grafana cloud)** 

Here are the steps to install Grafana on a Linux machine:

1. Add the Grafana GPG key to the system:

```
curl https://packages.grafana.com/gpg.key | sudo apt-key add -
```

1. Add the Grafana repository to the system:

```
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
```

1. Update the package index and install Grafana:

```
sudo apt update
sudo apt install grafana
```

1. Start the Grafana service:

```
sudo systemctl start grafana-server
```

1. Enable the Grafana service to start at boot:

```
sudo systemctl enable grafana-server
```

1. Check the status of the Grafana service:

```
sudo systemctl status grafana-server
```

*Example output from above command:*

![Screenshot 2023-05-07 at 23.29.05.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5576a768-ff87-4af3-98a0-ae7372c89ce0/Screenshot_2023-05-07_at_23.29.05.png)
