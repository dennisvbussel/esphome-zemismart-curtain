# ESPHome firmware for Zemismart Curtain
Custom ESPHome component for the Zemismart WiFi slide curtain (new design). 

Even though Tasmota has an integration for the slide curtain available, I would like to integrate this into a custom component for ESPHome since i'm a huge fan of the native API between Home Assistant and ESPHome.

Inspiration and fragments of the code came from fellow developers [iphong](https://github.com/iphong) and [brandond](https://github.com/brandond), please feel free to take a look at their repositories.

- [ESPHome custom components for Tuya curtain includes feedback and position support](https://github.com/iphong/esphome-tuya-curtain)
- [ESPHome support for cheap WiFi PIR sensors](https://github.com/brandond/esphome-tuya_pir)

## Features

* Automatically sends a heartbeat message every 10 seconds
* Controls open, close, stop and custom position (0 up to 100)
* Position feedback once the curtain motor stops
* Hass Services:
  * **get_status_report** returns the status of all dpids. (e.g. to retrieve position information after HA reboot)
  * **set_motor_normal** and **set_motor_reversed** sets the operation mode.
  * **send_command** can be used to manually send commands to the motor.

## Known Bugs

During development I observed strange behaviour when using the commands open, stop and close. The included crc appears to be incorrect however, the command is still executed correctly by the motor. Bug in the embedded software of the Tuya module perhaps?

Open:  
TX:  `55AA00060005010400010212`  
RX:  `55AA03070005010400010215` Included CRC = 0x15 calculated CRC = 0x16

Close:  
TX:  `55AA00060005010400010010`  
RX:  `55AA03070005010400010013` Included CRC = 0x13 calculated CRC = 0x14

Stop:  
TX:  `55AA00060005010400010111`  
RX:  `55AA03070005010400010114` Included CRC = 0x14 calculated CRC = 0x15



## Hardware Overview

The Zemismart slide curtain is driven by the ZZM79E-DT motor. After removing the back cover the Tuya TYWE1S module is exposed. The module is soldered on a small break-out PCB, connecting directly to the motor using a 6 pin 2 mm header. The datasheet for this module can be found here:  [TYWE1S Module Datasheet-Documentation-Tuya Developer](https://developer.tuya.com/en/docs/iot/wifie1smodule?id=K9605thnvg3e7)

<img src="https://github.com/dennisvbussel/esphome-zemismart-curtain/blob/main/Pictures/TYWE1S_Module.jpg" alt="TYWE1S_Module" style="zoom:20%;" />



Using the multimeter the following pinout is obtained (pinout table matches the picture):

| Pin Number : | Function: | Pin Number: | Function: |
| ------------ | --------- | ----------- | --------- |
| 1            | GND       | 2           | VCC       |
| 3            | GND       | 4           | 3V3       |
| 5            | U1RX      | 6           | U1TX      |

The exact curtain I ordered can be found here (link might be broken in the future):

[Zemismart New Design WiFi Curtain Motor Tuya Smart Life Customized Electric Curtains Track with RF Remote Alexa Echo Control|Smart Remote Control| - AliExpress](https://www.aliexpress.com/item/32933434297.html?spm=a2g0o.cart.0.0.1d983c00ympn48&mp=1)



## Serial Protocol

The first step was to analyse the traffic between the Tuya Module and the curtain motor using a logic analyser. All communication within this chapter is described from the Tuyas Module point of view. 

* TX = Tuya Module -> Motor MCU
* RX = Tuya Module <- Motor MCU

### Boot sequence:

**1) Detecting heartbeat**  
TX:  `55AA00000000FF`  
RX:  `55AA030000010003`  (0x00 indicates the first heartbeat)

**2) Querying product information**  
TX:  `55AA0001000000`  
RX:  `55AA0301002A7B2270223A222A2A2A2A2A2A2A2A2A2A2A2A2A2A2A2A222c2276223A22312E302E30222C226D223A307DB3`  
      {"p":"****************","v":"1.0.0","m":0}

**3) Querying working mode**  
TX:  `55AA0002000001`  
RX:  `55AA0302000004` (0x0000 indicates coordination mode with MCU)

**4) Querying Status (Async)**  
TX:  `55AA0008000007`  
RX:  `55AA03070005010400010216`  (dpid 1 - curtain control)  
RX:  `55AA0307000803020004000000001A`  (dpid 8 - motor direction)  
RX:  `55AA03070005050100010116`  (dpid 5 - motor direction)  
RX:  `55AA0307000507040001011B`  (dpid 7 - unknown)  
RX:  `55AA030700050A050001001E`  (dpid 10 - error report)  

**5) Reporting device network status (Wi-Fi has been configured but not connected to the router)**  
TX:  `55AA000300010205`  
RX:  `55AA0303000005`

**6) Reporting device network status (Wi-Fi has been configured and connected to the router.)**  
TX:  `55AA000300010306`  
RX:  `55AA0303000005`

**7) Reporting device network status (Wi-Fi has been connected to the router and the cloud.)**  
TX:  `55AA000300010407`  

**6) Detecting heartbeat**  
TX:  `55AA00000000FF`  
RX:  `55AA030000010104 `

### Use cases:

**Control  commands**

Open:  
TX:  `55AA00060005010400010212`  
RX:  `55AA03070005010400010215`  
RX:  `55AA0307000803020004000000647E ` (Reports position on stop)

Stop:  
TX:  `55AA00060005010400010111`  
RX:  `55AA03070005010400010114`  
RX:  `55AA03070008030200040000002B42 ` (Reports position on stop)

Close:  
TX:  `55AA00060005010400010010`  
RX:  `55AA03070005010400010013`  
RX:  `55AA0307000803020004000000001A ` (Reports position on stop)

Motor direction set normal operation:  
TX:  `55AA00060005050100010011`  
RX:  `55AA03070005050100010015`  

Motor direction set reversed operation:  
TX:  `55AA00060005050100010112`  
RX:  `55AA03070005050100010116`  

**Control by remote or pull curtain**

Open/Stop/Close:  
RX:  `55AA0307000507040001001A` (same response for open/stop/close)  
RX:  `55AA0307000803020004000000001A` (Reports position on stop)

Close + curtain blocked:  
RX:  `55AA0307000507040001001A`  
RX:  `55AA030700050A0500010220`  
RX:  `55AA0307000803020004000000001A` (Reports position on stop)

