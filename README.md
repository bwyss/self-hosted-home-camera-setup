# Self Hosted Home Camera Setup
### Ubuntu, CasaOS, Frigate, Home Assistant, Reolink, Google Coral TPU setup guide

![Screenshot-20241001081922-1498x919](https://github.com/user-attachments/assets/b62672c6-d024-4032-bd9b-48883350e02d)


### Requirements
Dedicated "server" PC with:
- Plenty of storage
- At least one available Mini PCIe or M.2 module motherboard slot
- Linux: 64-bit version of Debian 10 or Ubuntu 16.04 (up to 20.04), and an x86-64 or ARMv8 system architecture
- Python 3.6-3.9
- Camera (wired and PoE) - in my case REOLINK RLC-811A PoE IP Security Camera
- Ethernet Switch
- USB pen drive (for ubuntu installation)


### Overview
This setup offers:
- Self hoseting for security cameras
- Object detection
- Notifications
- Live view
- Access to media clips, recordings and snapshots 

DO NOT USE Ubuntu version > 20.04. Ubuntu 20.04 (desktop in my case) is needed because it is using a kernel version 5.15.0-67-generic which is compatible with the Edge TPU runtime and the TensorFlow Lite runtime software used for the Google Coral TPU hardware.

The Google Coral TPU chip is needed for all of the object detection that frigate is going to be doing on the video streams. Without this chip, the system will use excessive GPU for the processing. My motherboard has a Mini M.2 slot so I [purchased this chip](https://www.amazon.com/dp/B0CY2C6FV4?ref=ppx_yo2ov_dt_b_fed_asin_title).

I am using CasaOs for easy docker management (Frigate, Eclipse Mosquitto and Home Assistant will be install with CasaOS on top of ubuntu desktop)

I'm using [Reolink](https://www.amazon.com/gp/product/B09873G7X3/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&th=1) cameras with a [switch](https://www.amazon.com/gp/product/B076PRM2C5/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&th=1)

### Step 1 Install Ubuntu 20.4
Follow [this guide](https://ubuntu.com/tutorials/install-ubuntu-desktop#4-boot-from-usb-flash-drive) for the installation off of a USB drive

### Step 2 Install the PCIe driver and Edge TPU runtime
[Follow this guide](https://coral.ai/docs/m2/get-started/#1-connect-the-module)

### Step 3 Install CasaOS
```bash
wget -qO- https://get.casaos.io | sudo bash
```
Here is the [wiki docs](https://wiki.casaos.io/en/get-started), but all you need is to run that command ^^
At the end of the instalation it will tell you the IP address thatCasoOS will be running on, in my case: [http://10.0.0.32/](http://10.0.0.32/), you should be able to access this address from any device that is on your home network.

### Step 4 Install Frigate
CasaOS has a nifty app store, but it does not include Frigate, [so here is a guide for that](https://www.youtube.com/watch?v=y6YW1OvoDK4&t=204s).

I skipped the step for installing the Frigate config and just added my own after the installation. To add the Frigate config, go to the CasaOS home screen and click on files. Then navigate to DATA/AppData/frigate/config and open the config.yml file. Refer to the [Frigate docs](https://docs.frigate.video/) for configuring this for your needs, but here is a sample to get you started:

```bash
mqtt:
  enabled: false
detectors:
  coral:
    type: edgetpu
    device: pci
cameras:
  Driveway:
    enabled: true
    ffmpeg:
      hwaccel_args:
        - '-c:v:1'
        - h264_v4l2m2m
      inputs:
        # Get your camera IP address from the network settings in the camera app
        # Get the stream info from your camera docs e.g. h264Preview_01_main which is going to be different for every type of camera
        - path: 'rtsp://admin:YOURPASS@10.0.0.54:554/h264Preview_01_main' 
          roles:
            - detect
        - path: 'rtsp://admin:YOURPASS@10.0.0.54:554/h264Preview_01_sub'
          roles:
            - record
    detect:
      # Height and width can be found in your cameras app
      width: 640
      height: 360
      fps: 10
    snapshots:
      enabled: true
    record:
      enabled: true
    motion:
      threshold: 25
      contour_area: 100
      delta_alpha: 0.2
      frame_alpha: 0.2
    objects:
      track:
        - person
        - cat
        - dog
        - bird
        - car
        - bear
version: 0.14
```
You can not open the Frigate app in CasaOS and you should be able to see the camera(s) feeds. You can configure as many cameras as you have in the Frigate config.
If you don't see your camera feed you can check the docker logs by opening a terminal and following these steps:
``` bash
docker ps
```
You will get an output like this, which contains the docker image id:
``` ls
d33218b513e1 ghcr.io/blakeblackshear/frigate:stable
```

Next run:
``` bash
docker logs <YOURDOCKERIMAGEID>
```

Check the logs for errors or give the logs to chatGPT and debug as needed.

Once the Frigate config is happy you will see your camera feed.

### Step 5 Install Home Assistant w/ HACS & Frigate
Home Assistant will allow you to view your Frigate clips, recording and snapshots. Frigate can be added to Home Assistant using the HACS plugin manager. HACS can be added to HA using the "Add On Store" feature, but the Add On Store  is not available in the docker version of HA. So you will need to follow these steps:

In a terminal, navigate to the home assistant directory
```bash
cd /DATA/AppData/homeassistant/config
```
Then add HACS:
```bash
wget -O - https://get.hacs.xyz | bash -
```
Then reboot your server:
```bash
sudo reboot
```
Open CasaOS and open the Home Assistant app. 
- Go to Settings > Devices & Services and then click the Add Integration button.
- Use the search bar to look for "hacs". Click on HACS.
- Check everything (it’s optional) and click Submit.
- Open HACS and add Frigate

In Home Assistant go to settings > Device and Services, then Add Integration, seach for and add Frigate.

Now you can access Frigate in Media using Home Assistant
s
### Step 5 MQTT Notifications using the Mosquitto Broker and Frigate Notifications

In CasaOS install a new custom app and add the mosquitto docker compose file:

``` bash
name: practical_ofir
services:
  mosquitto:
    cpu_shares: 90
    command: []
    container_name: mosquitto
    deploy:
      resources:
        limits:
          memory: 15926M
    environment:
      - MQTT_PASSWORD=<yourPass>
      - MQTT_USERNAME=<yourName>
    hostname: mosquitto
    image: eclipse-mosquitto:latest
    ports:
      - target: 1883
        published: "1883"
        protocol: tcp
      - target: 9001
        published: "9001"
        protocol: tcp
    restart: unless-stopped
    volumes:
      - type: bind
        source: /DATA/AppData/mosquitto/data
        target: /mosquitto/data
      - type: bind
        source: /DATA/AppData/mosquitto/config
        target: /mosquitto/config
    devices: []
    cap_add: []
    network_mode: bridge
    privileged: false
x-casaos:
  author: self
  category: self
  hostname: ""
  icon: ""
  index: /
  is_uncontrolled: false
  port_map: "1883"
  scheme: http
  store_app_id: practical_ofir
  title:
    custom: mosquitto
```
Once mosquitto is installed and running, navigate to /DATA/AppData/mosquitto/config and create a mosquitto.conf file and populate it with:

``` bash
listener 1883
allow_anonymous true
password_file /mosquitto/config/passwd
```

In the future change allow_anonymous to true and extend this with a user name and password. Keep allow_anonymous false for testing.

Extend the frigate config with:
```batch
mqtt:
  host: <YourIPAddress>
  port: 1883
  #user: <yourUserName>
  #password: <yourPassword>
  client_id: frigate
  topic_prefix: frigate
```

In home assistant, add the MQTT integration and test as needed to make sure that it is able to listen to frigate topics.

In home assistant, add the [Frigate Notification blueprint](https://community.home-assistant.io/t/frigate-mobile-app-notifications-2-0/559732)

Add hoem assistant automations as needed based on frigate entities.

### Step 6 Remote Access

[tailscale](https://tailscale.com/) vpn
