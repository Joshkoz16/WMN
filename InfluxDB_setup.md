1. To install InfluxDB, the InfluxDB CLI client, and the InfluxDB Python library, run the following command:

```
sudo apt install influxdb influxdb-client python3-influxdb
```
2. To start the InfluxDB service, enable it to start at boot, and open the influx UI, run the following commands:

```
sudo systemctl start influxdb
sudo systemctl enable influxdb
influx
```
3. The InfluxDB interface UI should open and display the following:

```
Connected to http://localhost:8086 version 1.6.7~rc0
InfluxDB shell version: 1.6.7~rc0
> 
```

4. To create an admin user authentication into all databases, run the following command in the InfluxDB UI. (In this case, the username is 'admin' and the password is 'password'):

```
CREATE USER 'admin' WITH PASSWORD 'password' WITH ALL PRIVILEGES
```

5. To enable authentication in the config file, exit the InfluxDB UI with the command 'quit' and then run the following command:

```
sudo nano /etc/influxdb/influxdb.conf
```

6. Inside the file, go to the http section, set 'auth-enabled' to 'true' and save the file. The InfluxDB service will need to be restarted with the following command.

```
sudo systemctl restart influxdb
```

7. With authentication enabled, the InfluxDB service must be started with the following command where we give it the admin credentials made earlier:

```
influx -username admin -password password
```

8. To create a database inside the InfluxDB UI and show all current databses, run the following commands. (In this case, the database name is 'energy'):

```
CREATE DATABASE energy
SHOW DATABASES
```

9. At this point, databases can only be accessed with the admin user. To create a dedicated user with R/W access limited to this database run the following command:

```
CREATE USER 'energyuser' WITH PASSWORD 'password'
GRANT ALL ON energy to energyuser
exit
```

10. Writing data points to the InfluxDB database uses line protocol. Data written must follow the following format:

```
measurement, (tag_set) field_set (timestamp)
```

For example, if our measurement is 'power_info', the tag is 'sensor' and its value is 'motor1', and our field_sets are 'power_in' and 'power_out' with values '145' and '568', respectively, then the line protocol format would appear as follows:

```
power_info,sensor=motor1 power_in=145,power_out=568
```

11. To open the InfluxDB UI as 'energyuser', open the 'energy' database', and write the above data point, run the following commands:

```
influx -username energyuser -password password
USE energy
INSERT power_info,sensor=motor1 power_in=145,power_out=568
```

12. To read this data point from the database, we can use an SQL-like query with the following command:

```
SELECT * FROM power_info
```

The following should then be displayed:

```
name: power_info
time                power_in power_out sensor
----                -------- --------- ------
1696862040408915862 145      568       motor1
```
In this case, we did not specify a timestamp and InfluxDB automatically assigns one at the time it is entered into the database.

13. To show the measurements and tag keys, the following commands can be used:

```
SHOW MEASUREMENTS
SHOW TAG KEYS
```

Now we have Influx DB installed on our Raspberry Pi, a local database created, authentication enabled, and verified that data can be written into the database.
