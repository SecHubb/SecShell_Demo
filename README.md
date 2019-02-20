# Philly Security Shell Meetup Elastic Stack Demo

## Prerequisites:

To run this lab you will need both the docker engine and docker compose installed. If you can run "docker -v" and "docker-compose -v" and get a version number back, you should be good to go. If you see errors when you run the docker-compose file, you may just need to reboot.

---

## Starting up the Elastic Stack

To bring up Elasticsearch, Kibana, and Cerebro (a 3rd party management framework for the Elastic stack) navigate to the folder where you have cloned this respository. At the command line type

`docker-compose up`

and press enter. You should see a bunch of text scrolling as the the containers start up. After 30 seconds or so you should be able to access Kibana at http://localhost:5601 as well as the Cerebro management tool at http://localhost:9000. To connect Cerebro to the running Elasticsearch container type 
 
`http://elasticsearch:9200` 

in the Node address box on the main page, then press the Connect button. You should see a dashboard showing a single row with your running Elasticsearch node.

To shut down services, press control+C on the window where docker-compose was started. You can also type `docker-compose stop` from inside the repo folder in another terminal. To remove all the containers type `docker rm elasticsearch kibana cerebro` once the services are stopped.

Note: This docker compose file will write all data into folders in the /tmp directory on your machine so that if you bring the containers down and back up, the data will still be there. If you wish to save the data somewhere else, edit the docker-compose.yml file under the volumes section of the elasticsearch service. You must choose a location where docker will have permissions to write.

---

## Data Import

For this demonstration we will use a firewall log that was recorded from a VPS located on digital ocean. The data is present for the timeframe of roughly 2019-02-12 to 2019-02-18. To keep attackers interested, low-interaction honeypot services were set up for multiple ports. Traffic to port 2222 should be ignored as the honeypot was administered through that port. Also note that outbound traffic from the honeypot itself is present in the data, if you do want to filter outbound traffic remove `source_ip:104.248.50.195` from your searches and visualizations.

To import the ufw.log data into Kibana follow these steps:
 
1. Go to Kibana in your virtual machine by opening a browser and going to http://localhost:5601. Select the machine learning plugin on the left side of Kibana, then select the Data Visualizer tab at the top of the screen.
2. Click the button to "Select or drag and drop a file" and pick the ufw.log filefrom the repo. If you see an error at this stage just try it again, there seems to be a potential bug with the 7.0.0beta1 of Elasticsearch that may cause it to error out if analysis takes too long.
3. After "Analyzing data" the next screen will show some information, but it does not fully detect the fields correctly, so we must fix it. Select the "Override settings" button in the Summary section and in the Grok pattern window clear out what is there and paste the below statement, then press Apply. Paste the following in the grok pattern window:

```%{SYSLOGBASE} \[%{DATA}\] \[%{DATA:action}\] IN=(%{WORD:in})? OUT=(%{WORD:out})?( MAC=%{DATA:mac})? SRC=%{IP:source_ip} DST=%{IP:destination_ip} %{DATA} PROTO=%{WORD:protocol}( SPT=%{INT:source_port} DPT=%{INT:destination_port})?```
 
4. Kibana will now re-analyze the data with the fields parsed correctly. You should now see the file stats display with data parsed out cleanly. Hit the blue import button in the lower left of the screen to go to the next page.
5. In the Import data section on the next page, select the Advanced tab.
6. Type the name `ufw_logs` in the "Index name" field, ensuring the "Create index pattern" box is checked. This will be the name of the index the data is imported into.
7. You will need to replace the contents of the **Mappings** and **Ingest pipeline** window in its entirety with the code below. The Index settings window can be left as-is. (This code is only making 3 small adjustments from the default. In the ingest section we are adding "processors" to resolve the geoIP location and ASN from the data in the source_ip field, and in the mappings where are defining a geoip object with a nested field called location of type "geo_point" where the resolved geoIP location data can be stored.)
 
Place the following in the **mappings** window:

```
{
  "@timestamp": {
    "type": "date"
  },
  "action": {
    "type": "keyword"
  },
  "destination_ip": {
    "type": "ip"
  },
  "destination_port": {
    "type": "long"
  },
  "in": {
    "type": "keyword"
  },
  "geoip": {
        "properties": {
          "location": { "type": "geo_point" }
        }
  },
  "length": {
    "type": "long"
  },
  "logsource": {
    "type": "keyword"
  },
  "mac": {
    "type": "keyword"
  },
  "message": {
    "type": "text"
  },
  "out": {
    "type": "keyword"
  },
  "program": {
    "type": "keyword"
  },
  "protocol": {
    "type": "keyword"
  },
  "source_ip": {
    "type": "ip"
  },
  "source_port": {
    "type": "long"
  }
}
```
 
 
Place the following in the ingest pipeline window:
```
{
  "description": "Ingest pipeline created by file structure finder",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{SYSLOGBASE} \\[%{DATA}\\] \\[%{DATA:action}\\] IN=(%{WORD:in})? OUT=(%{WORD:out})?( MAC=%{DATA:mac})? SRC=%{IP:source_ip} DST=%{IP:destination_ip} LEN=%{INT:length} %{DATA} PROTO=%{WORD:protocol}( SPT=%{INT:source_port} DPT=%{INT:destination_port})?"
        ]
      }
    },
    {
      "date": {
        "field": "timestamp",
        "timezone": "UTC",
        "formats": [
          "MMM dd HH:mm:ss",
          "MMM  d HH:mm:ss"
        ]
      }
    },
    {
      "remove": {
        "field": "timestamp"
      }
    },
    {
      "geoip": {
        "field" : "source_ip"
      }
    },
    {
      "geoip": {
        "field" : "source_ip",
        "database_file": "GeoLite2-ASN.mmdb"
      }
    }
  ]
}
```

8. Once this is complete, click the blue **import** button and your data should be cleanly ingested. You will be able to view it by going to the Discover tab and selecting the name of the index (ufw_logs) you chose in the previous steps. You also need to select the correct timeframe in the time picker at the upper right of the window in the Discover tab. Select Feb 12th, 2019 to Feb 18th, 2019 inclusive, and the data should show up on the page once you hit the Refresh button.
9. To import the premade visualizations and dashboards, use the `dashboard.json` file included with this repo. Click on the management tab on the left side of Kibana, then choose **Saved Objects**. On the next screen click the import button then choose the text file you saved the JSON as to import it. *(You may receive a warning message in a yellow box saying "The following saved objects use index patterns that do not exist. Please select the index patterns you'd like re-associated with them. You can create a new index pattern if necessary." Use the drop-down under **new index pattern** to select the `ufw_logs` index and hit **confirm all changes**.)*

The data, visualizations, and dashboards should be imported with the prefix [Shell] and you can access them through the related tabs on the left side of the screen.
