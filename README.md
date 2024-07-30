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
   Copy the created Token ID, we will need it later for Telegraf config.
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
   Copy the created Token ID, we will need it later for Grafana config.
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

# Set up Openwrt to send statistics to Telegraf/InfluxDB
1. Login to Luci and go to `Statistics->Setup->Output plugins->Network->Configure`
   In the `Server Interface` section provide `Server Host` as `Telegraf-IP-address` and `Server Port` as `25827`
   ![obraz](https://github.com/user-attachments/assets/b8742665-3d44-4a5a-b0a1-c76210575745)
   (In my expample `Telegraf-IP-address` is `192.168.100.99`)


