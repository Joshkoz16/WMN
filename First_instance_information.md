This first instance used the iPerf3 tool between Rasp02 and Rasp03. The script that ran the tool and collected the data was on Rasp03. The InfluxDB databases that stored the collected data were on Rasp03.

Here is a list of all the needed information to use this instance:

(InfluxDB admin credentials)  
Username: admin  
Password: password  
To start InfluxDB, type the following command:  
```
influx -username admin -password password
```

(InfluxDB user credentials)  
Username: rasp03  
Password: password  
This user is assigned to the rasp03wmn database; the primary database used when testing this instance.  

(InfluxDB user credentials)  
Username: energyuser  
Password: energy  
This user is assigned to the energy database; the database used when testing if InfluxDB was properly installed and configured.  

Inside the InfluxDB database, here are some of the commands that can be used:  
To show all databases (must be authenticated in as admin):  
```SHOW databases```  
To show all users (must be authenticated in as admin):  
```SHOW users```   
To insert a data point:  
```INSERT (measurement),(tag_set) (field_value1),(field_value2) (timestamp)```  
Note: If no timestamp is given, InfluxDB Will assign the current time as the data point is entered in.  
To show different measurements inside the InfluxDB database:  
```SHOW measurements```  
To show different tag keys inside the InfluxDB database:  
```SHOW tag keys```  

