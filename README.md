**Grafana dashboard for Openwrt using Collectd and Influx (With Downsampling)**

# Introduction
   
In this instruction I will describe you how you can export your collectd statistics to InfluxDB and then display them in Grafana. This will allow you to get detailed statistics about your Openwrt (or any other system with collectd installed) performance. The graphs are almost realtime - the level of detail is 1 minute for last 24 hours. The statistics are downsampled what allows you to view information what was the system performance in the past (several month or years ago) while still keeping database size small. Of course level of details for statistics from one year ago is much lower (6 hours) that for the statistics for last 24 hours (1 minute). 
The solution that I describe has been created basing on [Collectd to InfluxDB v2 with downsampling using Tasks](https://cloud-infra.engineer/collectd-influxdb2-grafana-with-downsampling/). 

Side note - if you want to have simple and working solution use: [CGP](https://github.com/pommi/CGP). I am still using this and found it to be very versatile.

# Prerequisities
Before you start you need to prepare and install following aplications on your server hardware (I am using Raspberrypi4 for this task):

    - Influx 2.0 (probably it will work with version 1.8 also)
    - Influx 2.0 CLI
    - Telegraf
    - Grafana
    - Collectd

# InfluxdB configuration
1. Create org `my-org` (if it already exist skip this step)
   ```
   influx org create -n my-org
   ```
2. Download and Apply Influxdb template for Gociwd
   ```
   wget https://raw.githubusercontent.com/mstojek/gociwd/main/gociwd-influx-template.yml
   influx apply -o my-org -f gociwd-influx-template.yml 
   ```
3. Create 'collectd' user with write access to bucket 'collectd/autogen' bucket
   ```
   BUCKET=$(influx bucket list --hide-headers -n collectd/autogen | awk '{print $1}')
   influx auth create --org my-org --description 'collectd' --read-bucket $BUCKET --write-bucket $BUCKET
   ```
   Copy the created Token ID (called by me below `InfluxDB-Token-For-Collectd-User`), we will need it later for Telegraf config.
4. Create 'grafana' user with read access for the 'collectd/*' buckets
   ```
   BUCKET_AUTOGEN=$(influx bucket list --hide-headers -n collectd/autogen | awk '{print $1}')
   BUCKET_DAY=$(influx bucket list --hide-headers -n collectd/day | awk '{print $1}')
   BUCKET_WEEK=$(influx bucket list --hide-headers -n collectd/week | awk '{print $1}')
   BUCKET_MONTH=$(influx bucket list --hide-headers -n collectd/month | awk '{print $1}')
   BUCKET_YEAR=$(influx bucket list --hide-headers -n collectd/year | awk '{print $1}')
   BUCKET_TENYEAR=$(influx bucket list --hide-headers -n collectd/tenyear | awk '{print $1}')
   BUCKET_HUNDYEAR=$(influx bucket list --hide-headers -n collectd/hundyear | awk '{print $1}')
   influx auth create --org my-org --description 'grafana' --read-bucket $BUCKET_AUTOGEN --read-bucket $BUCKET_DAY --read-bucket $BUCKET_WEEK --read-bucket $BUCKET_MONTH --read-bucket $BUCKET_YEAR --read-bucket $BUCKET_TENYEAR  --read-bucket $BUCKET_HUNDYEAR
   ```
   Copy the created Token ID (called by me `Grafana-Token`), we will need it later for Grafana config.
# Setup Telegraf
1. Add stations counter to 'types.db' - Openwrt uses this type for counting WIFI connected stations
   ```
   cat /usr/share/collectd/types.db
   [...]
   # OpenWRT
   stations                value:GAUGE:0:256
   ```
2. Edit Telegraf configuration
   ```
   cat /etc/telegraf/telegraf.d/collectd.conf
   [[inputs.socket_listener]]
     service_address = "udp://:25827"
     data_format = "collectd"
     collectd_typesdb = ["/usr/share/collectd/types.db"]
     collectd_parse_multivalue = "split"

     [inputs.socket_listener.tags]
       bucket = "collectd"

   [[outputs.influxdb_v2]]
     urls = ["http://<InfluxdDB-IP-address>:8086"]
     token = "<InfluxDB-Token-For-Collectd-User>"
     organization = "my-org"
     bucket = "collectd/autogen"

     [outputs.influxdb_v2.tagpass]
       bucket = ["collectd"]
   ```
   Note that we use port 25827 as port 25826 is used by Collectd. You need also to fill proper 'InfluxdDB-IP-address' and 'InfluxDB-Token-For-Collectd-User'

3. Restart Telegraf service
   
   `sudo service telegraf restart`

# Set up Openwrt/Collectd to send statistics to Telegraf/InfluxDB
1. Login to Luci and go to `Statistics->Setup->Output plugins->Network->Configure`

   In the `Server Interface` section provide `Server Host` as `Telegraf-IP-address` and `Server Port` as `25827`
   ![obraz](https://github.com/user-attachments/assets/b8742665-3d44-4a5a-b0a1-c76210575745)
   
   (In my example `Telegraf-IP-address` is `192.168.100.99`)

2. If you prefer to edit config files instead of using Luci add following lines to your collectd conf file `/etc/collectd/collectd.conf`

   ```
   LoadPlugin network
   <Plugin network> 
   Server "<Telegraf-IP-address>" "25827"
   </Plugin>
   ```

# Setup Grafana
1. Setup Flux Data Source
   
   Open you Grafana address in browser `(http://<Grafana-IP-address>:3000)`

   Login to Grafana
     
   Navigate to `Home->Connections->DataSources->AddDataSource`

   - Enter name for `Data Source` (e.g `collectd`)

   - In section `Query language` 
     Choose `Flux`

   - In section `HTTP`
     Provide URL for your InfluxDB with proper port `http://<InfluxdDB-IP-address>:8086`

   - In section `Auth`
     Uncheck Basic auth (nothing should be chosen)

   - In section `InfluxDB details`
     As `Organization` provide `my-org`
     Provide `Grafana-Token` that we created above

   - Click "Save&Test"

   ![obraz](https://github.com/user-attachments/assets/25cef571-20a0-4bb9-9c07-2d6d7b96f0ff)

2. Import Grafana Dashboard

   - Copy Dashboard Template file [gociwd-grafana-template.json](https://raw.githubusercontent.com/mstojek/gociwd/main/gociwd-grafana-template.json) to your local hard disk

   - Go to `Grafana->Home->Dashboards->Import dashboard` and Drag and Drop [gociwd-grafana-template.json](https://raw.githubusercontent.com/mstojek/gociwd/main/gociwd-grafana-template.json) file to the Import Field.
  
   - Go to `Grafana->Home->Dashboards` and open new Dashboard `Openwrt Collectd Graph Panel (Flux)`
   
# Example statistics

- Load, Memory and CPU statistics
![obraz](https://github.com/user-attachments/assets/66e23c7c-7360-49a5-ac57-26fad7ffbbd4)

- Interfaces Traffic data
![obraz](https://github.com/user-attachments/assets/0dc80779-77d0-4f5d-a54d-e9977711e165)
![obraz](https://github.com/user-attachments/assets/4b1982bc-927e-46ca-b9bc-e91b7ab7f4bc)

- Client Traffic data ([Iptmon](https://github.com/oofnikj/iptmon) or [Nlbwmon2Collectd](https://github.com/mstojek/nlbw2collectd) required)
![obraz](https://github.com/user-attachments/assets/9afd8715-339d-4aef-a46f-d0057d9f949c)
![obraz](https://github.com/user-attachments/assets/5b7b3872-b1ae-494d-b826-a1b224ab5f78)

- WIFI, Connections, DNS and DHCP leases statistics
![obraz](https://github.com/user-attachments/assets/d1b21ac4-f36f-4eeb-a954-4ec2cf9332c6)

- More statistics can be easily added by creating new Charts in Grafana. By default all statistics collected by Collectd are send to InfluxDB.
 
# Data aggregation

1. Data is aggeregated over time. Following aggregations are set:
   - Daily - data resolution is 1 minute
   - Weekly - data resolution is 5 minutes
   - Monthly - data resolution is 30 minutes
   - Yearly - data resolution is 6 hours
   - 10 Year - data resolution is 1 day
   - 100 Year - data resolution is 1 week
   Original data send from collectd is store in `Collect/autogen` bucket
2. Example charts:
   - Hourly
     ![obraz](https://github.com/user-attachments/assets/43b50a9a-11ae-4f9d-90b6-4dc1e1f6e5fa)
   - Daily
     ![obraz](https://github.com/user-attachments/assets/f123b22b-f4ff-46f1-b6a6-852176cd48bc)
   - Monthly
     ![obraz](https://github.com/user-attachments/assets/21df2b0e-cd9d-49e5-b1ba-362e56962167)
   - 1 year ago in September
     ![obraz](https://github.com/user-attachments/assets/4425430e-4987-4c8a-beb3-76d2c1807496)
     
# Chart parameters
![obraz](https://github.com/user-attachments/assets/8e49807b-01b0-4f49-91ec-4f2b648f7673)
1. We have following chart parameters that can be choosen form drop-down menu
   - `Hostname` - we can collects statistics from several collectd instances on several hosts (e.g. if you have more than one OpenWRT router, or some Raspberrypi hosts etc...)
   - `Client` - if you have [Iptmon](https://github.com/oofnikj/iptmon) or [Nlbwmon2Collectd](https://github.com/mstojek/nlbw2collectd) installed you can filter `Client Traffic` statistics just for this client
   - `Interface` - you can filter `Interfaces Traffic` data for choosen interface
   - `Wifi-Interface` - you can filter `Wifi` data for choosen Wifi interface
   - `Aggregation Type` - available options are: `Min, Max, Mean`
      - Explanation: in default configuration collectd sends statistics every 30 seconds (sometimes it could be 10 seconds or other). If you choose in my chart "monthly aggregation" (last month data) we can not have data resolution to be 30 seconds. Instead this resolution is changed to 5 minutes (to save disk space). In the 5 minutes period we have ten 30 seconds periods. Now we need to chose what the aggregate point should show - maximum value from those ten perdios, average value or maybe minimum? It depends on data type. For example for `Traffic` you might be interested in max value, while for `Idle processor usage` we might be wondering what was the minimum value. Here you can choose what is interesting for you
   - `Retention policy` - this is just informative field you can not change this value, it shows what retention bucket has been chosed for displaying the data.

# Connect Null Values
In the Grafana charts you have an option named `Connect Null Values`. I prepared two Grafana templates:
- template where this value is set to `Threshold <1h` - this is file: [gociwd-grafana-template.json](https://github.com/mstojek/gociwd/blob/main/gociwd-grafana-template.json)

  This seetting Properly displays short term charts
  
  ![obraz](https://github.com/user-attachments/assets/de9547c0-33ae-4450-9366-970243209560)

  However the long term charts consist of dots that are not conected
  
  ![obraz](https://github.com/user-attachments/assets/04057aaf-d7d5-4217-a46e-a1980850c174)

     
- template where this value is set to `Always` - this is file:[gociwd-grafana-template_AllwaysConnectNullValues.json](https://github.com/mstojek/gociwd/blob/main/gociwd-grafana-template_AllwaysConnectNullValues.json)

  This setting incorrectly displays short term charts
  
  ![obraz](https://github.com/user-attachments/assets/b5d99358-9eed-40ce-b57e-480f55328035)

  Note that between 9:10 and 13:30 nothing happened on the system, this chart should be "Null" or zero at this time. Instead Grafana connected last point from ~9:10 whit first point at ~13:30

  However the long term data have pretty nice look
  
  ![obraz](https://github.com/user-attachments/assets/915bb5c0-6e33-45b3-b5a9-35297af747ed)

- I do not have any solution for this issue.

# References

1. Collectd Graph Panel - great tool to display collectd RRD files: [CGP](https://github.com/pommi/CGP)
2. My solution is an evolution of following system described at: [Collectd to InfluxDB v2 with downsampling using Tasks](https://cloud-infra.engineer/collectd-influxdb2-grafana-with-downsampling/)

   What I added is
   - rewriting everything to Flux (not good decision as Flux is in [maintenance mode](https://docs.influxdata.com/flux/v0/future-of-flux/) now....)
   - better handling of data downsampling
4. Small but important topic on how to properly calculate "total usage" counters: [Add previousN and nextN to range](https://github.com/influxdata/flux/issues/702)
5. Very smart solution for dynamically choosing right bucket basing on timespan choosen in Grafana: http://wiki.webperfect.ch/index.php?title=Grafana:_Dynamic_Retentions_%28InfluxDB%29
  
  




