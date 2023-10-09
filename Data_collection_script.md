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

Note: This script 'datascript03' accesses a local InfluxDB database 'rasp03wmn' with username 'admin' and password 'password'. Edit the script as necessary for your setup.

Save the script to the Raspberry Pi and allow it to be run as an executable with the following command:

```chmod +x datascript03```

