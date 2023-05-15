---
title: "Hands-on: Build your own Industrial IoT Demo - Part 2"
date: 2023-05-12 22:16:31 +03:00
tags: [Industrial IoT, Digital Manufacturing, Open Source]
description: How to build your own Industrial IoT solution
---

Let's continue where we left off, software configurations.

## **Software Setup**

### **Mosquitto MQTT Broker on IoT2040**

Mosquitto is ready to use in IoT2040 image as we mentioned before. If you are facing any issue with Mosquitto, check /etc/mosquitto/mosquitto.conf file and make sure it’s configured as below:

```yaml
listener 1883
allow_anonymous true
```

### **Node-Red on IoT2040**

Node-Red is also ready to use in IoT2040 image. We’ll start to build our own flow to collect XDK sensor data and several actions should be performed on ingested data before sending it to IoT Platform so let’s do it step by step:

1. Node-Red instance is available at http://<iot2040_ip_address>:1880
2. “MQTT In” node is needed to ingest XDK data. Add it to the flow and configure new MQTT broker as http://<iot2040_ip_address>:1883 and MQTT Topic as “raw/Acme/Milling/CNC_01/env_sensor”
    <figure>
    <img src="/handson-iiot-demo-part2/msg_payload.png" alt="Example Payload">
    <figcaption>Example Payload</figcaption>
    </figure>
3. When we check the incoming data, we’ll see raw temperature, humidity and pressure values in one message. As a first step we need to split those messages and send them separately. In order to do that, add “Split” node. “Object split” will be used in our case since we want to send each key/pair in individual message. Click “Copy key to” box and set the field as “msg.sensor” so our keys will be saved under msg.sensor. 
4. Now we split our messages but they are still going through same route. In order to differentiate the route of individual messages, we’ll use “Switch” node. Add Switch node to the flow and connect Split and Switch nodes. Set property field as “msg.sensor” and add three different route as below.
    <figure>
    <img src="/handson-iiot-demo-part2/msg_split.png" alt="Split Node Configuration">
    <figcaption>Split Node Configuration</figcaption>
    </figure>
5. Before starting pre-processing, we want to eliminate unnecessary load for IoT Platform so we only want to send the sensor value if it’s changed, it’s called report by exception. We have “rbe” node exactly for this job so we’ll add it into our flow and set the property as “msg.payload”. This node will check if new value is different from previous one and if yes, it’ll send it to next node. If not, message will be discarded.
6. Now we have separate messages in their own route so we can start to pre-process the messages. Here’s our list of task to perform: 1) Add timestamp for each message, 2) Reduce temperature decimal precision to 1, 3) Convert pressure unit to Pa to kPa and reduce decimal precision to 2. We’ll add three different “function” node to perform these tasks and add related codes to the individual nodes as below:

    ```jsx
    // Temperature
    msg.payload = {
        "time": new Date(),
        "measurement": parseFloat(msg.payload.toFixed(1)),
    }
    return msg;
    ```
    
    ```jsx
    // Humidity
    msg.payload = {
        "time": new Date(),
        "measurement": msg.payload,
    }
    return ms
    ```
    
    ```jsx
    // Pressure
    msg.payload = {
        "time": new Date(),
        "measurement": parseFloat((msg.payload/1000).toFixed(2)),
    }
    return msg;
    ```
7. As a last step, we want to combine multiple messages in one message before sending them to IoT platform in order to reduce message traffic. In our case XDK sends data every second but we’ll wait for 10 seconds and combine all the messages in one message and then release it. Add “Join” node into our flow and use below configuration:
    <figure>
    <img src="/handson-iiot-demo-part2/msg_join.png" alt="Join Node Configuration">
    <figcaption>Join Node Configuration</figcaption>
    </figure>
8. Finally our message is ready to be sent via MQTT. Add “MQTT out” node and configure 
Server: “http://<raspberrypi_ip_address>:1883”
Topic: “raw/Acme/Milling/CNC_01/env_temperature”
9. Don’t forget to deploy flow by clicking “Deploy” button on top-right corner.

Final flow should look like this:
<figure>
<img src="/handson-iiot-demo-part2/edge_flow.png" alt="Node-Red Flow on Edge">
<figcaption>Node-Red Flow on Edge</figcaption>
</figure>

### Docker on Raspberry Pi

We’ll use containerization to manage our different services in our IoT Platform.

1. Create a folder for our project named “iotdemo” and put below docker-compose.yml file in it. Check your user id with “id -u” command if it’s different than “1000”, change user configuration in docker-compose file accordingly. Replace username and password fields as well.

	```yaml
	version: '3.3'
	services:
	  grafana:
		image: grafana/grafana:latest
		user: "1000"
		container_name: grafana
		restart: unless-stopped
		ports:
		  - "3000:3000"
		environment:
		  - GF_SECURITY_ADMIN_USER=<your_username_here>
		  - GF_SECURITY_ADMIN_PASSWORD=<your_password_here>
		volumes:
		  - ./grafana/data:/var/lib/grafana
		networks:
		  - iotnetwork

	  nodered:
		image: nodered/node-red:latest
		user: "1000"
		container_name: nodered
		restart: unless-stopped
		ports:
		  - "1880:1880"
		volumes:
		  - ./node-red/data:/data
		networks:
		  - iotnetwork

	  timescaledb:
		image: timescale/timescaledb:latest-pg13
		user: "1000"
		container_name: timescaledb
		restart: unless-stopped
		ports:
		  - 5432:5432
		environment:
		  - POSTGRES_PASSWORD=<your_password_here>
		volumes:
		  - ./timescaledb/data:/var/lib/postgresql/data
		networks:
		  - iotnetwork

	  mosquitto:
		image: eclipse-mosquitto
		container_name: mosquitto
		user: "1000"
		restart: unless-stopped
		ports:
		  - "1883:1883"
		volumes:
		  - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf
		networks:
		  - iotnetwork

	networks:
	  iotnetwork:
	```

2. Create folder for Mosquitto in iotdemo folder with "mkdir mosquitto", create conf file in this folder with “sudo nano mosquitto.conf” and add below configurations:

	```yaml
	listener 1883
	allow_anonymous true
	```

3. run “docker-compose up -d” command in iotdemo directory to create docker containers for Node-Red, Mosquitto MQTT Broker, TimescaleDB and  Grafana. 
4. In case of need, you can restart the containers with “docker-compose restart” or stop all containers with “docker-compose down” commands.

### **Node-Red on Raspberry Pi**

We’ll again use Node-Red as a business logic layer for our IoT Platform. We’ll only need one additional node outside of default palette to write data to our database. In order to install “node-red-contrib-postgresql”:

1. Open the Node-Red editor by navigating to "http://<raspberry_pi_ip_address>:1880" in your web browser.
2. Click menu icon on top right and then “Manage palette”
3. Under “Install” tab, search for “node-red-contrib-postgresql” and click install.
4. “postgresql” node should be visible in the right panel. 

Now we can start to build our flow in our IoT Platform:

1. Add “MQTT In” node to ingest data from Edge Gateway. Set server as "http://<raspberry_pi_ip_address>:1883" and Topic as “raw/#”.

> We’ll use generalized rules which can be applied for all raw data therefore we’ll be able to collect all raw data from single ingestion point.
> 
2. In order to utilize postgresql node, we need to prepare the message accordingly. Below code first parse the MQTT topic to identify site, area, device and sensor information. Then it will extract timestamps and sensor values from the payload to create arrays. Finally create list of parameters to be used by query and create query. 

```jsx
// Split the MQTT topic string into separate components
var topicItems = msg.topic.split("/");
var site = topicItems[1];
var area = topicItems[2];
var device = topicItems[3];
var sensor = topicItems[4];

// Extract timestamps and values from payload
var values = [];
var times = [];
msg.payload.forEach(function (data) {
    times.push(data.time);
    values.push(data.measurement)
});

// Define parameters
msg.params = [times, site, area, device, sensor, values];

// Generate query
msg.query = "INSERT INTO iotraw (time, site, area, device, sensor, measurement) SELECT unnest($1::timestamp[]), $2, $3, $4, $5, unnest($6::double precision[])";

return msg;
```

3. Finally we can add “postgresql” node after function node and under Connection tab:
Host: http://<raspberry_pi_ip_address>
Port: 5432
Database: iotraw
SSL: false
and then under Security tab:
User: postgres
Password: your_password_here
4. Don’t forget to deploy flow by clicking “Deploy” button on top-right corner.

Final flow should look like this:
<figure>
<img src="/handson-iiot-demo-part2/iotplatform_flow.png" alt="Node-Red flow on IoT Platform">
<figcaption>Node-Red flow on IoT Platform</figcaption>
</figure>

### **TimescaleDB on Raspberry Pi**

TimescaleDB is a time-series database that is optimized for storing and querying large volumes of time-series data. We’ve already created TimescaleDB instance with our docker-compose. Now we’ll create simple database and table in it:

1. Connect TimescaleDB instance on Docker:

```bash
docker exec -it timescaledb psql -U postgres
```

2. Create a new database by running the following command:

```sql
CREATE DATABASE iotraw;
```

3. Connect to the new database by running the following command:

```sql
\c iotraw;
```

4. Create a regular PostgreSQL table to store the sensor data by running the following command:

```sql
CREATE TABLE iotraw (
  time TIMESTAMPTZ NOT NULL,
  site TEXT NOT NULL,
  area TEXT NOT NULL,
  device TEXT NOT NULL,
  sensor TEXT NOT NULL,
  measurement DOUBLE PRECISION NULL
);
```

5. Convert the regular table into a hypertable partitioned on the “time” column using the create_hypertable() function provided by Timescale:

```sql
SELECT create_hypertable('iotraw', 'time');
```

6. Exit the PostgreSQL server by running the following command:

```sql
\q
```

### **Grafana on Raspberry Pi**

Grafana is a popular open-source tool for creating dashboards and visualizations for time-series data. To install and configure Grafana on the Raspberry Pi, follow these steps:

1. Navigate to "http://<raspberry_pi_ip_address>:3000" in your web browser to access the Grafana interface.
2. Log in to Grafana using the username and password you configured in docker-compose.yml file
3. Click on “Menu” icon and then under “Connetions”, click “Connect data”. Search for “PostgreSQL” and click on icon. Finally click “Create a PostgreSQL data source” button.
4. Configure the PostgreSQL data source by entering the following information:
- Name: iotdemo
- Host: http://<raspberry_pi_ip_address>:5432
- Database: iotdemo
- User: postgres
- Password: your_password_here
- TLS/SSL Mode: disable
5. Click on the "Save & Test" button to test the connection to the TimescaleDB database.
6. Create a new dashboard by clicking on the "+" icon in the top-right corner of the screen and selecting "New dashboard". Click again “+ Add visualization” button.
7. Configure the graph by selecting the "iotdemo" data source, selecting the "iotraw" table, and configuring the query to retrieve the desired data. Query section has “builder” and “code” option. Choose “code” and paste below query to the editor.

```sql
SELECT time_bucket_gapfill('1 minute', "time")  AS "time", AVG(measurement) AS "value", sensor 
FROM iotraw 
WHERE time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
GROUP BY time_bucket_gapfill('1 minute', "time"), sensor 
ORDER BY time ASC
```

8. Save the dashboard by clicking on the "Save" button in the top-right corner of the screen.
9. Now we should see the dashbard panel as below:
<figure>
<img src="/handson-iiot-demo-part2/grafana_dashboard.png" alt="Grafana Dashboard">
<figcaption>Grafana Dashboard</figcaption>
</figure>
Congratulations! You have now built an industrial IoT demo using the Bosch XDK, Siemens IoT2040, Raspberry Pi 4, and a variety of open-source software tools.

Since our demo was very small scale, we did not extensively consider the performance metrics of the chosen open-source solutions. Our primary focus was on selecting well-known and easy-to-implement components that allowed us to quickly kickstart our project.

## Conclusion

As an advocate of open-source software in the industrial environment, I strongly emphasize the importance of its increasing usage. The utilization of open-source solutions plays a crucial role in driving innovation, flexibility, and cost-effectiveness.

Our exploration into building an Industrial IoT solution using open-source technologies has only scratched the surface. Moving forward, we will continue to examine our existing architecture and identify potential bottleneck areas. Our aim is to optimize the system by replacing certain components with more suitable alternatives. I encourage you to stay connected and follow my upcoming posts as we delve deeper into these topics.

By embracing open-source solutions, industrial businesses can unlock new avenues of growth, enhance operational efficiency, and drive ongoing advancements in their respective industries. The power of open collaboration and the limitless possibilities offered by open-source technologies make it an indispensable choice for organizations seeking to thrive in today's competitive landscape.

Thank you for joining me on this journey, and I look forward to sharing further insights and discoveries with you in the future.
