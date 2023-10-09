To display Grafana, go to the following URL:

```10.106.92.144:3000```

The needed credentials (at least for rasp03 where tests have been conducted) is:

``
username = 'admin'
password = 'password'
```

To add a new connection, click on the menu on the left-hand side of the screen and click "Connections" then "Add new connection".

Scroll down and select "InfluxDB" and click on "Add new data source" in the upper right-hand corner.

Give your new database a name (in my case, "InfluxDB-1"). Enter the URL (http://localhost:8086), enable Basic auth and enter the InfluxDB username and password credentials (in my case, "admin" and "password"), as well as the database name (in my case, "energy") and the database's username and password (in my case, "admin" and "password").

Set HTTP Method to "GET"

Click "Save & test" and you should receive the output "datasource is working. 1 measurement found"

Return to the left-hand menu and select "Dashboards", we can select "New" then "New dashboard" and choose our display method. Here, we will choose "New visualization". Select the name of the database that was just linked. Underneath the "Panel Title" box will be a collection of options for query "A".

Select "select measurement" and set it to "power_info"
Select "field(value)" and set it to "power_in"

Scroll down and click "Add query" to add query "B".

Select "select measurement" and set it to power_info"
Select "field(value)" and set it to "power_out"

