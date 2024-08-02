# Introduction
   
In this instruction I will describe you how you can export your collectd statistics to InfluxDB and then display them in Grafana.

# Prerequisities
Before you start you need to prepare and install following aplications on your server hardware (I amd using Raspberrypi4 for this task):

    - Influx 2.0
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

- Client Traffic data (Iptmon or Nlbwmon2collectd  required)
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

1. We have following chart parameters that can be choosen form drop-down menu
   - `Hostname` - we can collects statistics from several collectd instances on several hosts (e.g. if you have more than one OpenWRT router, or some servers on Raspberrypi), here you can choose statistic form one collectd instance
   - `Client` - if you have Iptmon or [Nlbwmon2Collectd](https://github.com/mstojek/nlbw2collectd) installed you can filter `Client Traffic` statistics just for this client
   - `Interface` - you can filter `Interfaces Traffic` data per choosen interface
   - `Wifi-Interface` - you can filter `Wifi` data per choosen Wifi interface
   - `Aggregation Type` - 


  
  




