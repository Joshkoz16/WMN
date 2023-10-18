# Setup Script -> InfluxDB -> Grafana

The following will cover all the steps needed to configure the Raspberry Pi for data collection, storage, and visualization using iPerf3, InfluxDB, and Grafana.

## Install InfluxDB, create users, create databases, and write data points into the database:  
To install InfluxDB, the InfluxDB CLI client, and the InfluxDB Python library, run the following command:  
```
sudo apt install influxdb influxdb-client python3-influxdb
```  
To start the InfluxDB service, enable it to start at boot, and open the influx UI, run the following commands one at a time:  
```
sudo systemctl start influxdb
sudo systemctl enable influxdb
influx
```  
The InfluxDB interface UI should open and display the following:  
```
Connected to http://localhost:8086 version 1.6.7~rc0
InfluxDB shell version: 1.6.7~rc0
> 
```  
To create an admin user authentication into all databases, run the following command in the InfluxDB UI. (In this case, the username is 'admin' and the password is 'password'):  
```
CREATE USER admin WITH PASSWORD 'password' WITH ALL PRIVILEGES
```  
To enable authentication in the config file, exit the InfluxDB UI with the command 'quit' and then run the following command:  
```
sudo nano /etc/influxdb/influxdb.conf
```  
Inside the file, go to the http section, set 'auth-enabled' to 'true' and save the file. The InfluxDB service will need to be restarted with the following command:  
```
sudo systemctl restart influxdb
```  
With authentication enabled, the InfluxDB service must be started with the following command where we give it the admin credentials made earlier:  
```
influx -username admin -password password
```  
To create a database inside the InfluxDB UI and show all current databses, run the following commands. (In this case, the database name is 'energy'):  
```
CREATE DATABASE energy
```
To confirm we've created the database, enter the command:  
```
SHOW DATABASES
```
You should see the following output:  
```
name: databases
name
----
_internal
energy
```
At this point, databases can only be accessed with the admin user. To create a dedicated user with R/W access limited to this database enter the following commands one at a time:  
```
CREATE USER 'energyuser' WITH PASSWORD 'password'
GRANT ALL ON energy to energyuser
exit
```  
Writing data points to the InfluxDB database uses line protocol. Data written must follow the following format:  
```
measurement, (tag_set) field_set (timestamp)
```  
For example, if our measurement is 'power_info', the tag is 'sensor' and its value is 'motor1', and our field_sets are 'power_in' and 'power_out' with values '145' and '568', respectively, then the line protocol format would appear as follows:  
```
power_info,sensor=motor1 power_in=145,power_out=568
```  
To open the InfluxDB UI as 'energyuser', open the 'energy' database', and write the above data point, run the following commands one at a time:  
```
influx -username energyuser -password password
USE energy
INSERT power_info,sensor=motor1 power_in=145,power_out=568
```  
To read this data point from the database, we can use an SQL-like query with the following command:  
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
*Note: In this case, we did not specify a timestamp and InfluxDB automatically assigns one at the time it is entered into the database. For this project, the automatically assigned time stamp is the one that has been used. If a different timestamp is needed, InfluxDB has documentation on their website on the format the timestamp needs to be in.*  
Now we have Influx DB installed on our Raspberry Pi, a local database created, authentication enabled, and verified that data can be written into the database.  

## Add a script to run iPerf3 command and write output to InfluxDB database:  
The following script can be used to provide necessary parameters and authentication for InfluxDB, run the iperf3 command, parse the iperf3 output for the proper data, and then write that data into the InfluxDB database:  
```
#!/usr/bin/python3

# import InfluxDBClient class and subprocess module
from influxdb import InfluxDBClient
import subprocess

# set parameters for the InfluxDBClient
client = InfluxDBClient(host='localhost',
                        port=8086,
                        username='admin',
                        password='password')

iperf_command = ['iperf3', '-c', '10.106.94.202', '-t', '1']  # set iperf3 parameters without a specific duration

# run indefinitely or until user stops. 
while True:
     ping_process = subprocess.Popen(iperf_command, stdout=subprocess.PIPE, text=True)    # runs a subprocess using the iperf3 command
     ping_output, _ = ping_process.communicate()     # gets the standard output of the ping_process command
     lines = ping_output.strip().split('\n')     # splits the output of the iperf command into a list of lines
     lines = lines[6:-3]      # removes the unneeded lines

     # to parse lines and get bitrate
     for line in lines:
          columns = line.split()     # takes each line and splits it into columns
          bitrate = columns[6]     # takes the 7th column and assigns it to bitrate
          print('Bitrate:', bitrate)  # print bitrate for debugging
          line = f'iperf,pi=rasp03 ping={bitrate},test=2'     # creates a literal for the data that will be written to the database
          client.write([line], {'db':'rasp03wmn'}, 204, 'line')     # use influxDB client to write 'line' to the database

# close influxDB client
client.close()
```

*Note: This script 'datascript' accesses a local database 'rasp03wmn' with username 'admin' and password 'password'. Edit the script as necessary for your setup.*  
*Note: This script runs the iPerf3 tool for 1 second before ending it and sending the output to the database. This is what the iPerf output will look like:*  
```
Connecting to host 10.106.94.250, port 5201
[  5] local 10.106.92.144 port 39908 connected to 10.106.94.250 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  7.11 MBytes  59.6 Mbits/sec    0    431 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-1.00   sec  7.11 MBytes  59.6 Mbits/sec    0             sender
[  5]   0.00-1.05   sec  6.55 MBytes  52.2 Mbits/sec                  receiver

iperf Done.
```
The script takes the '59.6' from the 7th line, removes the rest of the text, and saves the value to the database.  
Save the script to the Raspberry Pi and allow it to be run as an executable with the following command:  
```
sudo chmod +x datascript
```  
## Set up Grafana to query InfluxDB for data and display on Grafana:  

Set up the apt-key used to authenticate packages:  
```
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```
Add the Grafana apt repository:  
```
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/g
rafana.list
```
Then install Grafana with the following commands:
```
sudo apt-get update
sudo apt-get install -y grafana
```
Execute the following commands to configure Grfana to start automatically:  
```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server
```
You can also start Grafana with the following command:  
```
sudo /bin/systemctl start grafana-server
```
To check if Grafana is running, enter the following command:  
```
sudo lsof -i -n
```
You should see the command 'grafana' running.  
To display Grafana, go to the following URL:  
```
10.106.92.144:3000  // The IP address will be the address of the pi you have set up the script on. You can also enter 'localhost:3000'
```
By default, the user credentials are:  
```
username = admin
password = admin
```
When entered, Grafana will prompt you to update your password. For consistency, the following credentials have been used on all pis:
```
username = admin
password = password
```
* To add a new connection, click on the menu on the left-hand side of the screen and click "Connections" then "Add new connection"  
* Scroll down and select "InfluxDB" and click on "Add new data source" in the upper right-hand corner  
* Give your new database a name (in this case, "InfluxDB-1"). Enter the URL (http://localhost:8086), enable Basic auth, and enter the InfluxDB username and password credentials (in this case, "admin" and "password"), as well as the database name (in this case, "energy") and the database's username and password (in this case, "admin" and "password")  
* Set HTTP Method to "GET"  
* Click "Save & test" and you should receive the output "datasource is working. 1 measurement found"  
* Return to the left-hand menu and select "Dashboards", we can select "New" then "New dashboard" and choose our display method. Here, we will choose "Add visualization"  
* Select the name of the database that was just linked (in this case, "InfluxDB-1"). Underneath the "Panel Title" box will be a collection of options for query "A"  
* Click "select measurement" and set it to "power_info"  
* Click "field(value)" and set it to "power_in"  
* Scroll down and click "Add query" to add query "B"  
* Click "select measurement" and set it to power_info"  
* Click "field(value)" and set it to "power_out"
* Click "Save" in the upper right-hand corver and save the Dashboard name as "raspwmntest". *Note: if on rasp04, for example, save the dashboard as "rasp04test")*
* Choose the folder you wish to save the dashboard into and click "save"
* Grafana will save the dashboard and take you to the dashboard home page where your new panel will be displayed.

*Note: I entered the following additional data points into my energy database through the Influx UI:*  
``` 
1696862040408915862 145      568       motor1
1696866923096249602 145      568       motor1
1696866931209139817 135      523       motor1
1696866958507029333 146      570       motor1
1696866970652751866 102      500       motor1
1696866987873066167 195      592       motor1
```
You can enter these datapoints yourself with the following commands (*Note: enter each line in one at a time:*)    
```
INSERT power_info,sensor=motor1 power_in=145,power_out=568
INSERT power_info,sensor=motor1 power_in=145,power_out=568
INSERT power_info,sensor=motor1 power_in=135,power_out=523
INSERT power_info,sensor=motor1 power_in=146,power_out=570
INSERT power_info,sensor=motor1 power_in=102,power_out=500
INSERT power_info,sensor=motor1 power_in=195,power_out=592
```
If you type the following command:  
```
select * from power_info
```
You should now see the following data in your database:  
```
1697639303904942864 145      568       motor1
1697641732676634432 145      568       motor1
1697641732687244288 145      568       motor1
1697641732701039922 135      523       motor1
1697641732711470503 146      570       motor1
1697641732723807191 102      500       motor1
1697641733856462233 195      592       motor1
```

Your dashboard setup window should now look like this if you entered the same data points in (*Note: you may have to click on the refresh icon in the upper right-hand corner*:  
![Dashboard setup](https://github.com/Joshkoz16/WMN/blob/main/energy%20database%20test.png)

You can then click "Apply" in the upper right-hand corner to save this dashboard panel  

## All credentials, information, and commands used in first instance of data collection:  
This first instance used the iPerf3 tool between Rasp02 and Rasp03. The script that ran the tool and collected the data was on Rasp03. The InfluxDB databases that stored the collected data were on Rasp03.

Here is a list of all the needed information to use this instance:

**InfluxDB admin credentials:**  
Username: **admin**  
Password: **password**  
*Note: to start InfluxDB, type the following command:*  
```
influx -username admin -password password
```

**InfluxDB user credentials:**  
Username: **rasp03**  
Password: **password**  
*Note: this user is assigned to the rasp03wmn database; the primary database used when testing this instance*  
```
influx -username rasp03 -password password
```

**InfluxDB user credentials:**  
Username: **energyuser**  
Password: **password**  
*Note: this user is assigned to the energy database; the database used when testing if InfluxDB was properly installed and configured*  
```
influx -username energyuser -password password
```

**Grafana user credentials:**  
URL: **10.106.02.144:3000**  
Username: **admin**  
Password: **password**  

Inside the InfluxDB database, here are some of the commands that can be used:  

The SHOW command can be used to display the following:  
```
CONTINUOUS, DATABASES, DIAGNOSTICS, FIELD, GRANTS, MEASUREMENT, MEASUREMENTS, QUERIES, RETENTION, SERIES, SHARD, SHARDS, STATS, SUBSCRIPTIONS, TAG, USERS
```
To insert a data point:  
```
INSERT (measurement),(tag_set) (field_value1),(field_value2) (timestamp)
```  
*Note: If no timestamp is given, InfluxDB Will assign the current time as the data point is entered in.*  

To select a database to use:  
```
use <database name>
```
To delete a database, run the 'use' command to go in the database and then run:  
```
drop database <database name>
```

## Go from powered off Pi to data collection:

Power on Raspberry Pis named rasp2 and rasp3. For the purposes of this demonstration, rasp2 is the server Pi and rasp3 is the client Pi.  
Open a Terminal window on rasp2 and start the iPerf server with the command:
```
iperf3 -s
```
If you get an error stating, "error - unable to start listener for connections: Address already in use", this is becuase iperf3 is started at boot. You can stop iperf3 with the following steps.  
To see all commands running on the pi:  
```
sudo lsof -i -n
```
Look for 'iperf3' and note its PID number (the second column). Then run the command:  
```
sudo kill -9 <insert PID number here>
```
Then, reenter the 'lsof' command to double check that iperf3 has been killed.  
Now you can reenter the 'iperf3 -s' command to start the server on rasp2. You should see the following output:  
```
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```
On rasp3, open a Terminal window and enter the command:
```
ls
```
This will list all files and folders in the current directory /home/rasp3. Look for 'datascript' and enter this command to run the script:  
```
./datascript
```
You should begin to see the script's output in the rasp3 terminal window in the following format:  
```
['[  5]   0.00-1.00   sec  7.60 MBytes  63.7 Mbits/sec    0             sender', '[  5]   0.00-1.07   sec  6.46 MBytes  50.7 Mbits/sec                  receiver']
Bitrate: 63.7
Bitrate: 50.7
['[  5]   0.00-1.00   sec  7.38 MBytes  61.9 Mbits/sec    0             sender', '[  5]   0.00-1.08   sec  6.80 MBytes  52.8 Mbits/sec                  receiver']
Bitrate: 61.9
Bitrate: 52.8
```
On the rasp2 Terminal window, you should begin to see an output in the following format:  
```
Accepted connection from 10.106.92.144, port 39618
[  5] local 10.106.94.202 port 5201 connected to 10.106.92.144 port 39626
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  6.25 MBytes  52.4 Mbits/sec                  
[  5]   1.00-1.08   sec   566 KBytes  58.2 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.08   sec  6.80 MBytes  52.8 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.106.92.144, port 39628
[  5] local 10.106.94.202 port 5201 connected to 10.106.92.144 port 39636
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  3.11 MBytes  26.1 Mbits/sec                  
[  5]   1.00-2.00   sec  0.00 Bytes  0.00 bits/sec
```
Open up the Grafana dashboard you set up previously and if you have the graph properly set up you should see data being recorded in live time like this:  
![Dashboard setup](https://github.com/Joshkoz16/WMN/blob/main/iperf3%20ping%20updating%20example.png)
