# Setup Script -> InfluxDB -> Grafana

The following will cover all the steps needed to configure the Raspberry Pi for data collection, storage, and visualization using iPerf3, InfluxDB, and Grafana.

## Install InfluxDB, create users, create databases, and write data points into the database:  
To install InfluxDB, the InfluxDB CLI client, and the InfluxDB Python library, run the following command:  
```
sudo apt install influxdb influxdb-client python3-influxdb
```  
To start the InfluxDB service, enable it to start at boot, and open the influx UI, run the following commands:  
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
CREATE USER 'admin' WITH PASSWORD 'password' WITH ALL PRIVILEGES
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
SHOW DATABASES
```  
At this point, databases can only be accessed with the admin user. To create a dedicated user with R/W access limited to this database run the following command:  
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
To open the InfluxDB UI as 'energyuser', open the 'energy' database', and write the above data point, run the following commands:  
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
In this case, we did not specify a timestamp and InfluxDB automatically assigns one at the time it is entered into the database.  
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

iperf_command = ['iperf3', '-c', '10.106.94.250', '-t', '1']  # set iperf3 parameters without a specific duration

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

Note: This script 'datascript03' accesses a local database 'rasp03wmn' with username 'admin' and password 'password'. Edit the script as necessary for your setup.
Note: This script runs the iPerf3 tool for 1 second before ending it and sending the output to the database. This is what iPerf outputs:  
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
chmod +x datascript03
```  
## Set up Grafana to query InfluxDB for data and display on Grafana:  
To display Grafana, go to the following URL:  
```
10.106.92.144:3000
```
The needed credentials (at least for rasp03 where tests have been conducted) is:  
```
username = 'admin'
password = 'password'
```  
* To add a new connection, click on the menu on the left-hand side of the screen and click "Connections" then "Add new connection"  
* Scroll down and select "InfluxDB" and click on "Add new data source" in the upper right-hand corner  
* Give your new database a name (in my case, "InfluxDB-1"). Enter the URL (http://localhost:8086), enable Basic auth and enter the InfluxDB username and password credentials (in my case, "admin" and "password"), as well as the database name (in my case, "energy") and the database's username and password (in my case, "admin" and "password")  
* Set HTTP Method to "GET"  
* Click "Save & test" and you should receive the output "datasource is working. 1 measurement found"  
* Return to the left-hand menu and select "Dashboards", we can select "New" then "New dashboard" and choose our display method. Here, we will choose "New visualization" * * Select the name of the database that was just linked. Underneath the "Panel Title" box will be a collection of options for query "A"  
* Select "select measurement" and set it to "power_info"  
* Select "field(value)" and set it to "power_in"  
* Scroll down and click "Add query" to add query "B"  
* Select "select measurement" and set it to power_info"  
* Select "field(value)" and set it to "power_out"  

Note: I entered the following data points into my energy database through the Influx UI:  
``` 
1696862040408915862 145      568       motor1
1696866923096249602 145      568       motor1
1696866931209139817 135      523       motor1
1696866958507029333 146      570       motor1
1696866970652751866 102      500       motor1
1696866987873066167 195      592       motor1
```  
Your dashboard setup window should now look like this if you entered the same data points in:  
![Dashboard setup](https://github.com/Joshkoz16/WMN/blob/f784ca1c202d662292238f58f00196dda0f8c4a6/Screenshot%202023-10-09%20105827.png)

You can then click "Apply" in the upper right-hand corner to save this dashboard panel  

## All credentials, information, and commands used in first instance of data collection:  
This first instance used the iPerf3 tool between Rasp02 and Rasp03. The script that ran the tool and collected the data was on Rasp03. The InfluxDB databases that stored the collected data were on Rasp03.

Here is a list of all the needed information to use this instance:

**InfluxDB admin credentials:**  
Username: **admin**  
Password: **password**  
*Note:* to start InfluxDB, type the following command:  
```
influx -username admin -password password
```

**InfluxDB user credentials:**  
Username: rasp03  
Password: password  
*Note:* this user is assigned to the rasp03wmn database; the primary database used when testing this instance  
```
influx -username rasp03 -password password
```

**InfluxDB user credentials:**  
Username: **energyuser**  
Password: **password**  
*Note:* this user is assigned to the energy database; the database used when testing if InfluxDB was properly installed and configured  
```
influx -username energyuser -password password
```

Inside the InfluxDB database, here are some of the commands that can be used:  

To show all databases (must be authenticated in as admin):  
```
SHOW databases
```  
To show all users (must be authenticated in as admin):  
```
SHOW users
```   
To insert a data point:  
```
INSERT (measurement),(tag_set) (field_value1),(field_value2) (timestamp)
```  
*Note:* If no timestamp is given, InfluxDB Will assign the current time as the data point is entered in.  

To show different measurements inside the InfluxDB database:  
```
SHOW measurements
```  
To show different tag keys inside the InfluxDB database:  
```
SHOW tag keys
```  
