# RaspPiMesh  
The purpose of this file is so serve as a description of the setup and testing Tyler Nguyen did over Summer '23.  
The contents of this file were copied from Tyler's Github. For further information on the contents of this file, please contact him.  

## RaspPi setup:

Update the Raspberry Pi  
```
sudo apt-get update && sudo apt-get upgrade -y
```
Install gedit and iperf  
```
sudo apt-get install gedit
sudo apt install -y iperf3
```
Create script  
```
touch boot.sh
gedit boot.sh
```
Edit the file to contain the following. Note that the IP address's last digit should vary. For example, 192.168.1.1 and 192.168.1.2  
```
#!/bin/sh
sudo iw dev wlan1 interface add mesh0 type mp mesh_id MYMESHID
sudo iw mesh0 set type mp
sudo iw dev mesh0 set channel 4
sudo ifconfig wlan1 down
sudo ifconfig mesh0 up
sudo ip addr add 192.168.1.3/24 dev mesh0
```
Copy script to bin directory  
```
sudo cp boot.sh /usr/local/bin
```
Make the file executable by typing the following code into a terminal  
```
sudo chmod +x /usr/local/bin/boot.sh
```
Create unit file  
```
sudo gedit /etc/systemd/system/boot.service
```
Edit unit file  
```
[Unit]
Description=Raspberry Pi Mesh

Wants=network.target
After=syslog.target network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/boot.sh
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```
Change permissions  
```
sudo chmod 640 /etc/systemd/system/boot.service
```
Reload file  
```
sudo systemctl daemon-reload
```
Enable script at startup  
```
sudo systemctl enable boot
```
Go to dhcpcd  
```
sudo nano /etc/dhcpcd.conf
```
Disable wpa_supplicant by inserting commands  
```
denyinterfaces wlan1
denyinterfaces mesh0
```
Reboot  
```
sudo reboot
```
The next step will go into turning one of the Raspberry Pis into a server which will then go into testing.  
 ## Server.sh  
This section turns one of the Raspberyr Pis into a server for the iPerf3 command.  
Create script  
```
touch server.sh
gedit server.sh
```
Edit the file  
```
#!/bin/sh
iperf3 -s
```
Copy script to bin directory  
```
sudo cp server.sh /usr/local/bin
```
Make the file executable by type the following code into a terminal  
```
sudo chmod +x /usr/local/bin/server.sh
```
Create unit file  
```
sudo gedit /etc/systemd/system/server.service
```
Edit unit file  
```
[Unit]
Description= Raspberry Pi Server

Wants=network.target
After=syslog.target network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/server.sh
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```
Change permissions  
```
sudo chmod 640 /etc/systemd/system/server.service
```
Reload file  
```
sudo systemctl daemon-reload
```
Enable script at startup  
```
sudo systemctl enable server
```
Reboot  
```
sudo reboot
```
## Testing Part 1  
This section will go over a few commands to test speed and bitrate.  
For bitrate, on the Raspberry Pi that isn't the server, use the following command. In this scenario, the Pi with the assigned IP address 192.168.1.2 is the server.  
```
iperf3 -c 192.168.1.2
```
For speed, use the following ping command to test the speed between nodes.  
```
ping 192.168.1.2
```
For mpath, use the following command to show the path the mesh takes when communicating with one another.  
```
sudo iw dev mesh0 mpath dump
```
For station, use this command to see the signal as well as bitrate.  
```
sudo iw dev mesh0 station dump
```
### Notes:  
For each of the commands listed above, you can add ">" at the end of each of these commands to insert data into a file. In the example below, the data from the ping command will go into a new file named "test".  
```
ping 192.168.1.2 > test
```
You can specify how many packets the ping command runs for using the "-c" followed by a number. In the example below, the ping command will run for 20 packets.  
```
ping 192.168.1.3 -c 20
```
You can increase the rate at which the ping collects data using the "-i" followed by a number. The fastest rate of data collection for the ping command is 200ms. This commmand can also be applied to the iperf3 commands, except that command doesn't have the 200ms limitation. In the example below, the ping command collects data every 0.2 seconds.  
```
ping 192.168.1.3 -i .2
```
If you want the ping command to only display time, use the following command. It is important to note that the last line of data should be disregarded.  
```
ping 192.168.1.2 | grep -Po '[0-9.]*(?= ms)'
```
You can specify how long the iperf3 command runs for using the "-t" followed by a number. In this examply, the iperf3 command will run for 20 seconds.  
```
iperf3 -c 192.168.1.2 -t 20
```
You can use the following command to only display bitrate. It's important to note that the last two lines of data should be disregarded.  
```
iperf3 -c 192.168.1.2 | grep -Po '[0-9.]*(?= Mbits/sec)'
```
You can add "-f [format]" onto the iperf3 command to change units. For consistency reasons, I added in "m" so that the command will always print in mbits/sec.  
```
iperf3 -c 192.168.1.4 -f m
```
## Testing Part 2  
This section goes over an executable file that can be used to collect data efficiently. Note that this script will need to be edited for each experiment.  
Create the script. It is named "datacollect" in this example.  
```
touch datacollect.sh
gedit datacollect.sh
```
Edit the file to contain the following. Note that the file destination can be changed. For example, ex1test1ping can be edited to ex1test2ping after a trial.  
```
#!/bin/sh
ping 192.168.1.4 -c 90 | grep -Po '[0-9.]*(?= ms)' > ex1test1ping
iperf3 -c 192.168.1.4 -t 90 -f m | grep -Po '[0-9.]*(?= Mbits/sec)' > ex1test1iperf
sudo iw dev mesh0 station dump > ex1test1stat
sudo iw dev mesh0 mpath dump > ex1test1mpath
```
Make the file executable.  
```
sudo chmod +x datacollect.sh
```
Execute the file via the following.  
```
sh datacollect.sh
```
