This file will go over everything needed to install the influxDB process on a Raspberry Pi, create and use local databases, and link the database to a real time display in Grafana.

1. To install InfluxDB, the InfluxDB CLI client, and the InfluxDB Python library, run the following commands:

sudo apt install influxdb influxdb-client python3-influxdb
sudo systemctl start influxdb
sudo systemctl enable influxdb

2. To open InfuxDB, run the following command:

influx

3. 
