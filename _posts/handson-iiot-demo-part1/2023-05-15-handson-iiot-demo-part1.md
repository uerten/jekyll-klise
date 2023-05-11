---
title: "Hands-on: Build your own Industrial IoT Demo"
date: 2022-12-28 11:58:47 +03:00
tags: [Industrial IoT, Open Source]
description: How to build your own Industrial IoT solution
image: ""
---


Even though there are various Industrial IoT solutions on the market, developing your own solution from scratch can offer significant advantages in terms of flexibility, control, and cost-effectiveness. Also, it provides a deeper understanding of the complete architecture of Industrial IoT systems and their specific requirements. In this article, we will dive into the practical aspects and explore the process of creating our own Industrial IoT demo, covering every layer from the field level up to the visualization layer. By following this guide, you will gain valuable insights into both the hardware and software setup, enabling you to build a customized stack using open-source solutions. Let's embark on this hands-on journey to unleash the true potential of Industrial IoT.

## Use Case

In this demonstration, we will focus on a specific use case as an assumption: visualizing environmental data from an equipment in our factory. The primary goal is to monitor and analyze the impact of the environment on our manufacturing process. To accomplish this, we will deploy a smart sensor that will collect real-time environmental data directly from the equipment. This data will then be transmitted to our IoT Platform through an edge gateway. 

## **Solution Architecture**

In our Industrial IoT solution, we have designed the following solution architecture to efficiently collect, process, store, and visualize sensor data:

### **Smart Sensor and Edge Gateway Setup**

To collect and transmit sensor data, we will employ **[Bosch XDK](https://www.xdk.io/)** smart sensor. This device will gather the necessary data, which will be sent to the Edge Gateway using the MQTT protocol.

### **Data Pre-processing and IoT Platform Integration**

To optimize the raw sensor data, we will leverage the power of Node-RED for efficient data pre-processing on the Edge Gateway. Node-RED will handle data formatting and prepare it for further analysis. After pre-processing, the data will be transmitted to the IoT Platform via MQTT.

### **Data Storage and Visualization**

For secure and organized storage of the processed data, we will utilize TimescaleDB. Node-RED will write the transformed data into the database, facilitating easy retrieval and analysis.

To visualize and gain insights from the collected data, we will implement Grafana. Grafana will connect to TimescaleDB, enabling us to fetch the required data and generate visually appealing dashboards. These dashboards will provide valuable insights into the environmental impact on our manufacturing process, aiding in data-driven decision-making.

### Unified Namespace

It’s also worth mentioning that we will utilize the Unified Namespace, term coined by Walker Reynolds (4.0 Solutions), for our MQTT topic structure to ensure a standardized and organized approach to data management and retrieval in our Industrial IoT solution. Here are a few reasons why we opt for a unified namespace:

1. **Consistency:** By using a consistent namespace format, we establish a uniform structure for identifying and locating sensor data. This consistency makes it easier to understand and manage the data across different components of the solution.
2. **Scalability:** As the solution expands with additional sensors or equipment, a unified namespace allows for seamless integration of new devices. The namespace can accommodate multiple sensors within the same environment or across different locations, ensuring scalability without sacrificing organization.
3. **Hierarchy and Context:** The namespace structure, "raw/Acme/Milling/CNC_01/env_sensor" as an example for our demo, provides a hierarchical representation of the data. It includes relevant information about the site (Acme), the specific site area (Milling), the equipment (CNC_01), and the sensor type (env_sensor). This hierarchy adds contextual information to the data, making it easier to locate and understand.
4. **Searchability and Filtering:** A well-defined namespace enables efficient data search and filtering. With a unified structure, it becomes simpler to query and extract specific subsets of data based on criteria such as equipment type, location, or sensor category. This enhances data retrieval and analysis capabilities.
5. **Ease of Maintenance:** With a standardized namespace, managing and maintaining the data becomes more streamlined. It simplifies tasks such as data organization, data migration, and system updates, as all components follow a consistent naming convention.

<figure>
<img src="/handson-iiot-demo-part1/sol_arch.png" alt="Solution Architecture">
<figcaption>Solution Architecture</figcaption>
</figure>

### Requirements

For the successful implementation of this demo, we will require specific hardware and software components. Here is a breakdown of the essential items:

**Required Hardware:**

- Sensor: Bosch XDK
- Edge Gateway: Siemens IOT2040
- Platform Server: Raspberry Pi 4

**Required Software:**

- Containerization: Docker
- MQTT Broker: Mosquitto
- Business Logic: Node-Red
- Database: TimescaleDB
- Visualization: Grafana

By ensuring that you have these hardware and software components ready, you will be fully prepared to follow along with the demo and experience a seamless implementation of your Industrial IoT solution.

## **Hardware Setup**

### **Siemens IoT2040 as an Edge Gateway**
<figure>
<img src="/handson-iiot-demo-part1/iot2040.jpg" alt="Siemens IoT2040 Edge Gateway">
<figcaption>Siemens IoT2040 Edge Gateway</figcaption>
</figure>
Siemens IoT2040 is an industrial-grade edge gateway that can be used to collect sensor data from field and send it to the cloud. Apart from IoT2040 device, you can use any Linux based edge device to implement our solution. The following steps are needed to configure the IoT2040:

1. Start by burning the **[example image](https://support.industry.siemens.com/cs/document/109741799/downloads-for-simatic-iot20x0?lc=en-ww)** onto an SD card. Connect the SD card to the Siemens IoT2040 device. For detailed instructions on setting up the IoT2040, refer to **[this post](https://help.ubidots.com/en/articles/2046638-setting-up-the-siemens-simatic-iot2000)**.
2. The example image comes with pre-installed Node-Red and Mosquitto MQTT broker, eliminating the need for extensive software setup.
3. Once the device is successfully connected to your network, take note of the assigned IP address. You will require this information for the subsequent sensor device configuration.

### **Bosch XDK as a Field Sensor**
<figure>
<img src="/handson-iiot-demo-part1/bosch_xdk.jpg" alt="Bosch XDK Programmable Smart Sensor">
<figcaption>Bosch XDK Programmable Smart Sensor</figcaption>
</figure>
Bosch XDK is a programmable IoT sensor that is equipped with a full range of MEMS (micro-electromechanical system) sensors such as Accelerometer, Magnetometer, Gyroscope, Humidity/Temperature/Pressure Sensor, Acoustic Noise Sensor, Digital Light Sensor. XDK has its own workbench to program it in C or in more high level language [Eclipse Mita](https://www.eclipse.org/mita/platforms/xdk110/)

The following steps are needed to configure the XDK sensor:

1. Install the Bosch XDK Workbench software on your computer. You can download it from **[the official website](https://www.bosch-connectivity.com/products/cross-domain/cross-domain-developement-kit/downloads/)**
2. Workbench has already example projects that you can utilize for your needs. This is my version to send environmental data over MQTT in **[Github](https://github.com/uerten/bosch-xdk-senddataovermqtt)**. Download and import the project to Workbench. You only need to update “source/AppController.h” file for WLAN, SNTP and MQTT configuration
3. WLAN and SNTP configurations will depend on your specific setup, set them accordingly. The MQTT broker address should be set to http://<iot2040_ip_address>:1883, and the MQTT topic should be set as "raw/Acme/Milling/CNC_01/env_sensor".
4. Ensure that the XDK device is connected and visible in the Workbench. Once you have finished configuring the settings, click the "Flash" button located in the top left panel. This will build the project and then flash the program onto the XDK device.
5. Once the flashing process is complete, your XDK device is ready to transmit environmental data to the MQTT broker specified in the configuration.

### **Raspberry Pi 4 as an IoT Platform**
<figure>
<img src="/handson-iiot-demo-part1/raspberrypi4.jpg" alt="Raspbbery Pi 4">
<figcaption>Raspbbery Pi 4</figcaption>
</figure>
To implement our IoT platform, we have chosen the Raspberry Pi 4. While it is feasible to implement our software components on any Linux devices or servers, we specifically opted for the Raspberry Pi to highlight the possibility of utilizing modest, compact devices. Follow these step-by-step instructions for the hardware setup:

1. Install the Raspbian operating system on the Raspberry Pi. You can use the **[Raspberry Pi Imager](https://www.raspberrypi.com/software/)** tool to conveniently download and burn the relevant Raspbian image onto an SD card.
2. Next, install Docker on the Raspberry Pi by following the instructions provided **[here](https://linuxhint.com/install_docker_raspberry_pi-2/)**.
3. After installing Docker, proceed to install Docker Compose on the Raspberry Pi by following the steps outlined **[here](https://linuxhint.com/install-docker-compose-raspberry-pi/)**.

Before proceeding with the software setup, let's take a short break. When you're ready, we can move on to part 2.