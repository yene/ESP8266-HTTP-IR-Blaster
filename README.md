# ESP8266-HTTP-IR-Blaster

ESP8266 Compatible IR Blaster that accepts HTTP commands for use with services like Amazon Echo

![irblaster](image.jpg)

Supported Signals
--------------
NEC, Sony, Panasonic, JVC, Samsung, Sharp, Coolix, Dish, Wynter, Roomba, RC5/RC6, RAW

Hardware
--------------

![wiring](wiring.png)

- ESP8266 NodeMCU Board
- 2N2222 Transistor
- IRR1 = IR Receiver TSOP38238 (or the bit worse VS1838B)
- R1 = Resistor (I use 1k Ω)
- R2 = Resistor (I use 33 Ω)
- L1 = IR LED

> Resistor values depend on your IR LED, use http://ledcalc.com/ to calculate your resistors. [issue 12](https://github.com/mdhiggins/ESP8266-HTTP-IR-Blaster/issues/12)

> If you can send IR codes but your device does not recognize them, check that your transistor is connected correctly.

Drivers
--------------
Install the NodeMCU drivers for your respective operating system if they are not autodetected

https://www.silabs.com/products/mcu/Pages/USBtoUARTBridgeVCPDrivers.aspx

Setup
--------------
1. Install [Arduino IDE](https://www.arduino.cc/en/main/software)
2. Install [ESP8266 Arduino Core](https://github.com/esp8266/Arduino)
3. Install the following libraries from the Arduino IDE [Library Manager](https://www.arduino.cc/en/Guide/Libraries): `ESP8266WebServer` `ESP8266WiFi` `ArduinoJson` `WiFiManager`
4. Manually install the [IRremoteESP8266 library](https://github.com/markszabo/IRremoteESP8266)
5. Load the `IRController.ino` blueprint from this repository
6. Upload blueprint to your ESP8266. Monitor via serial at 115200 baud rate
> If you get an espcomm_upload_mem error you have selected the wrong board or wrong upload speed.
7. Device will boot into WiFi access point mode initially with SSID `IRBlaster Configuration`, IP address `192.168.4.1`. Connect to this and configure your access point settings using WiFi Manager
8. If your router supports mDNS/Bonjour you can now access your device on your local network via the hostname you specified (`http://hostname.local:port/`)
> If you cannot find your device look in the routers IP table.
9. Forward whichever port your ESP8266 web server is running on so that it can be accessed from outside your local network
10. Create an [IFTTT trigger](https://cloud.githubusercontent.com/assets/3608298/21918439/526b6ba0-d91f-11e6-9ef2-dcc8e41f7637.png) using the Maker channel using the URL format below. Make sure you use your external IP address and not your local IP address or local hostname

Server Info
---------------
You may access basic device information at `http://xxx.xxx.xxx.xxx:port/` (webroot)

Capturing Codes
---------------
Your last scanned code can be accessed via web at `http://xxx.xxx.xxx.xxx:port/last` or via serial monitoring over USB at 115200 baud. Most codes will be recognized and displayed in the format `A90:SONY:12`. Make a note of the code displayed in the serial output as you will need it for your maker channel URL. If your code is not recognized scroll down the JSON section of this read me.

> If you receive no data swap the pins on your IR Receiver.

Simple URL
--------------
For sending simple commands such as a single button press, or a repeating sequence of the same button press, use the logic below. This is unchanged from version 1.
Parameters
- `pass` - password required to execute IR command sending
- `code` - IR code such as `A90:SONY:12`
- `pulse` - (optional) Repeat a signal rapidly. Default `1`
- `pdelay` - (optional) Delay between pulses in milliseconds. Default `100`
- `repeat` - (optional) Number of times to send the signal. Default `1`. Useful for emulating multiple button presses for functions like large volume adjustments or sleep timer
- `rdelay` - (optional) Delay between repeats in milliseconds. Default `1000`
- `out` - (optional) Set which IRsend present to transmit over. Default `1`. Choose between `1-4`. Corresponding output pins set in the blueprint. Useful for a single ESP8266 that needs multiple LEDs pointed in different directions to trigger different devices

Example:
`http://xxx.xxx.xxx.xxx:port/msg?code=A90:SONY:12&pulse=2&repeat=5&pass=yourpass`

JSON
--------------
For more complicated sequences of buttons, such a multiple button presses or sending RAW IR commands, you may do an HTTP POST with a JSON object that contains an array of commands which the receiver will parse and transmit. Payload must be a JSON array of JSON objects. Password should still be specified as the URL parameter `pass`.

Parameters
- `data` - IR code data, may be simple HEX code such as `"A90"` or an array of int values when transmitting a RAW sequence
- `type` - Type of signal transmitted. Example `"SONY"`, `"RAW"`, `"Delay"` or `"Roomba"` (and many others)
- `length` - (conditional) Bit length, example `12`. *Parameter does not need to be specified for RAW or Roomba signals*
- `pulse` - (optional) Repeat a signal rapidly. Default `1`. *Sony based codes will not be recognized unless pulsed at least twice*
- `pdelay` - (optional) Delay between pulses in milliseconds. Default `100`
- `repeat` - (optional) Number of times to send the signal. Default `1`. *Useful for emulating multiple button presses for functions like large volume adjustments or sleep timer*
- `rdelay` - (optional) Delay between repeats in milliseconds. Default `1000`
- `khz` - (conditional) Transmission frequency in kilohertz. Default `38`. *Only required when transmitting RAW signal*
- `out` - (optional) Set which IRsend present to transmit over. Default `1`. Choose between `1-4`. Corresponding output pins set in the blueprint. Useful for a single ESP8266 that needs multiple LEDs pointed in different directions to trigger different devices.

3 Button Sequence Example JSON
```
[
    {
        "type":"nec",
        "data":"FF827D",
        "length":32,
        "repeat":3,
        "rdelay":800
    },
    {
        "type":"nec",
        "data":"FFA25D",
        "length":32,
        "repeat":3,
        "rdelay":800
    },
    {
        "type":"nec",
        "data":"FF12ED",
        "length":32,
        "rdelay": 1000
    }
]
```

Raw Example
```
[
    {
    "type":"raw",
    "data":[2450,600, 1300,600, 700,550, 1300,600, 700,550, 1300,550, 700,550, 700,600, 1300,600, 700,550, 700,550, 700,550, 700],
    "khz":38,
    "pulse":3
    }
]
```

Multiple LED Setup
--------------
If  you are setting up your ESP8266 IR Controller to handle multiple devices, for example in a home theater setup, and the IR receivers are in different directions, you may use the `out` parameter to transmit codes with different LEDs which can be arranged to face different directions. Simply wire additional LEDs to a different GPIO pin on the ESP8266 in a similar fashion to the default transmitting pin and set the corresponding pin to the `irsend1-4` objects created at the top of the blueprint. For example if you wired an additional LED to the GPIO0 pin and you wanted to send a signal via that LED instead of the primary, you would modify irsend2 in the blueprint to `IRsend irsend2(0)` corresponding to the GPIO0 pin. Then when sending your signal via the url simply add `&out=2` and the signal will be sent via irsend2 instead of the primary irsend1.

Default mapping
- irsend1: GPIO4
- irsend2: GPIO0
- irsend3: GPIO12
- irsend4: GPIO13
- irrecv: GPIO5

Force WiFi Reconfiguration
---------------
Set GPIO13 to ground for force a WiFi configuration reset

JSON and IFTTT
--------------
While the JSON functionality works fine with a command line based HTTP request like CURL, IFTTT's maker channel is not as robust.
To send the signal using the IFTTT Maker channel, simply take your JSON payload and remove spaces and line breaks so that entire packet is on a single line, then added it to the URL using the `plain` argument.

Sample URL using the same 3 button JSON sequence as above
```
http://xxx.xxx.xxx.xxx:port/json?pass=yourpass&plain=[{"type":"nec","data":"FF827D","length":32,"repeat":3,"rdelay":800},{"type":"nec","data":"FFA25D","length":32,"repeat":3,"rdelay":800},{"type":"nec","data":"FF12ED","length":32,"rdelay":1000}]
```

Roku
--------------
The Roku device supports sending commands via an API to simulate remote button presses over HTTP, but only allows connections via a local IP address. This blueprint supports sending these commands and acts as a bridge between IFTTT/Alexa to control the Roku with basic commands

Roku commands require 3 parameters that can be sent as a Simple URL or part of a JSON collection. Parameters include:
- `data` - [Roku code](https://sdkdocs.roku.com/display/sdkdoc/External+Control+Guide)
- `type` - Type of signal transmitted. Must be set to `roku`
- `ip` - Local IP address of your Roku device

Example Roku command to simulate pressing play button on a Roku with local IP `10.0.1.3`
```
http://xxx.xxx.xxx.xxx:port/msg?pass=yourpass&type=roku&data=keypress/play&ip=10.0.1.3
```
