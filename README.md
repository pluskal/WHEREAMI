 # WHERE AM I
WHERE AM I is a add-on for home assistant and allows you to determine the presence in rooms and areas in the room. For this purpose, it uses BLE beacons or mobile phones with a BLE beacon simulator. Tested with https://play.google.com/store/apps/details?id=net.alea.beaconsimulator&hl=en&gl=US. 

 ## SETUP
- You can add this repository to Add-on Store or copy folder to /addons on home assistant via SSH or Samba 
- Open the Home Assistant frontend
- Go to the Supervisor panel
- On the top right click the add-on store.
- On the top right overflow menu, click the "Reload" button.
- You should now see a new card called "Local add-ons" that lists new add-on.
- Click on your add-on to go to the add-on details page
- Install your add-on
- On the top, click the "Configuration".
    - Set your MQTT broker IP address and credentials.
    - Set your mongo string for your MongoDB. MongoDB is not available as addon in Home Assistant. You can use your own mongoDB server or try MongoDB Atlas which also offer free small MongoDB server.
    - You can change default DB name and default auto discovery prefix of homeassistant. More information about auto discovery prefix in https://www.home-assistant.io/docs/mqtt/discovery/.
- Now you can start addon in "Info" section.
- Addon is available on home assistant address with port 3000. You can use http://homeassistant.local:3000 or http://YOURIP:3000.
- After first access it can take a moment for full loading of all things.
- For now you can access interface via home assistant address with port 3000 or map this page in dashboard as web page card. Ingress is unavailable in this moment because of malfunctioning routing via ingress hash URL.

## BEFORE YOU START
- First you need to place several RSSI sensors around the house. The sensors are listed below. I recommend placing the sensors at a higher height, ideally against the ceiling. 
- On settings page, you can add your floor plan image to the system.
- First thing you need to add are rooms, the rooms can be added on the rooms page.
- You can then add your sensors to the system. There is an automatic detection feature on the sensors page that can help you find the sensor.
- When you have a sensor in your system, BLE devices will appear in the device list section of the device page. 
- Select the device to be monitored and use the scan page to learn fingerprints in the room. Then select your device in the statistics component and transfer the unused fingerprints to a set of workouts or fingerprint tests. If you have any training fingerprints, press the RETRAIN MODEL button.
- You should now see your device's predicted location on the device page.
- The system shares information about the location of your devices, presence in the rooms and a list of devices in the rooms with HA, where they appear as text and motion sensors. 

## SCANNING
When scanning, the areas must be selected so that they do not overlap. Personally, I recommend starting by scanning the corners and centers of the rooms. When scanning, I recommend normal movement or standing and turning in place to eliminate body shadows.

Subsequently, for example, on the basis of test fingerprints, it is possible to fine-tune the accuracy in areas with a lower determination success. Tuning is done by scanning the fingerprints for the place.


## SENSORS BASED ON ESP32 with ESPHome
ESPHome is popular solution and is available directly for Home Assistant. Information about instalation is on page https://esphome.io/guides/getting_started_hassio.html. After that you need sensor configuration which will be described below.

### SENSOR BASE
First of all you need specify your ESP32 sensor. For this purpose is `esphome` component. You specify name for your sensor, platform which is ESP32 for now and board type. Board type is string which you can find on https://docs.platformio.org/en/latest/platforms/espressif32.html#boards. More information about this component https://esphome.io/devices/esp32.html.

### WiFi
This core ESPHome component sets up WiFi connections to access points for you and you can find more info on https://esphome.io/components/wifi.html. 

### MQTT
The MQTT Client Component sets up the MQTT connection to your broker and is currently required for ESPHome to work. In most cases, you will just be able to copy over the MQTT section of your Home Assistant configuration. In this place you specify broker IP and credentials. More information on https://esphome.io/components/mqtt.html.

### BLE scanner part
This part of configuration is responsible for collecting of RSSI values in range and sending this value to MQTT. This part is based on ESPHome components ESP32 BLE tracker and text sensor.

```
esp32_ble_tracker:
  on_ble_advertise:
        then:
          - lambda: |-
              std::string val = "{";
              val += "\"address\":\"" + x.address_str() + "\"";
              val += ",\"rssi\":" + esphome::to_string(x.get_rssi());
              val += ",\"name\":\"" + esphome::to_string(x.get_name()) + "\"";
              val += ",\"uuid\":\"";
              if(x.get_ibeacon())
                val += x.get_ibeacon().value().get_uuid().to_string();
              val += "\"";
              val += "}";
              id(ts).publish_state(val);
              
  scan_parameters:
    active: true
    interval: 40ms
    window: 40ms

text_sensor:
  - platform: template
    name: "BLE Scanner 1"
    id: ts
```
### Optional PIR or other motion sensor
Optionaly you can add binary motion sensor. For this purpose is used GPIO Binary Sensor component from ESPHome. More information on https://esphome.io/components/binary_sensor/gpio.html.


### COMPLETE EXAMPLE:
Final configuration file will be like configuration file below.
```
esphome:
  name: %%YOURSENSORNAME%%
  platform: ESP32
  board: esp-wrover-kit

wifi:
  ssid: %%YOURSSID%%
  password: %%YOURWIFIPASSWORD%%


logger: # Enable logging
ota: # Enable Over The Air update

esp32_ble_tracker:
  on_ble_advertise:
        then:
          - lambda: |-
              std::string val = "{";
              val += "\"address\":\"" + x.address_str() + "\"";
              val += ",\"rssi\":" + esphome::to_string(x.get_rssi());
              val += ",\"name\":\"" + esphome::to_string(x.get_name()) + "\"";
              val += ",\"uuid\":\"";
              if(x.get_ibeacon())
                val += x.get_ibeacon().value().get_uuid().to_string();
              val += "\"";
              val += "}";
              id(ts).publish_state(val);
              
  scan_parameters:
    active: true
    interval: 40ms
    window: 40ms

text_sensor:
  - platform: template
    name: %%YOURBLESENSORNAME%%
    id: ts
    
binary_sensor:
  - platform: gpio
    pin: GPIO13
    name: %%YOURPIRSENSORNAME%%
    device_class: motion
    
mqtt:
  broker: %%YOURBROKERIP%%
  username: %%YOURMQTTUSERNAME%%
  password: %%YOURMQTTPASSWORD%%
```