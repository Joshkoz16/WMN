The following links will cover all the steps needed to configure the Raspberry Pi for data collection, storage, and visualization using iPerf3, InfluxDB, and Grafana.

To install InfluxDB, create users, create databases, and write data points into the database, [click here](InfluxDB_setup.md)

To add a script that will run the iPerf3 command and write the output bitrate to the InfluxDB database, [click here](Data_collection_script.md)

To set up Grafana to query the InfluxDB database for its data and display it on the Grafana dashboard, [click here](Grafana_setup.md)