Twinkly private protocol details
================================

This page describes hardware, modes of operation and some private procols or algorithms used by Twinkly application.


My hardware
-----------

I have model TW105S-EU. That's 105 RGB LED model from 2017.

Hardware consists of two circuit boards:

- Module ESP-01 with microcontroller ESP8266 by Espressif Systems.
- Custom-made LED driver module

API exposes these details:

- Product version: 2
- Hardware version: 6
- Flash size: 16
- LED Type: 6
- LED Version: 1
- Product code: TW105SEUP06


Firmware info
-------------
Firmware can be upgraded over the network. I have actually used strings from the firmware to find secret keys, encryption algorithms and some API calls that I haven't seen on the network. It consists of two files. First image format is according to https://github.com/espressif/esptool in version: 1.

I have seen these two versions only so this page describes its behaviour:

- 1.99.20
- 1.99.24
- 2.0.22-mqtt
- 2.1.0


Device name
-----------

Device name is used to announce SSID if it operates in AP mode, or to select device in the application. By default consists of prefix **Twinkly_** and uppercased unique identifier derived from MAC address. It can be read or changed by API.


Modes of network operation
--------------------------

Hardware works in two network modes:

- Access Point (AP)
- Station (STA)

AP mode is default - after factory reset. Broadcasts SSID made from `device name`_. Server uses static IP address 192.168.4.1 and operates in network 192.168.4.0/24. Provides DHCP server for any device it joins the network.

To switch to STA mode hardware needs to be configured with SSID network to connect to and encrypted password. Rest is simple API call through TCP port 80 (HTTP).

Switch from STA mode back to AP mode is as easy as another API call.

http://41j.com/blog/2015/01/esp8266-access-mode-notes/


WiFi password encryption
------------------------

1. Generate encryption key

   1. Use secret key: **supersecretkey!!**
   2. get byte representation of MAC adress of a server and repeat it to length of the secret key
   3. xor these two values

2. Encrypt

   1. Use password to access WiFi and pad it with zero bytes to length 64 bytes.
   2. Use rc4 to encrypt padded password with the *encryption key*

3. Encode

   Base64 encode encrypted string.


Typical Device Handshake
------------------------
1. UDP Broadcast Discovery
2. http GET gestalt (get device info)
3. http POST login (generate authentication_token) 
4. http GET verify login
5. http GET status

-- Firmware check
6. http GET firmware version
7. http update firmware (if firmware is out of date)

After Device Handshake
------------------------
8. http POST mode (off, demo, rt, movie)
9. UDP data stream OR Upload full movie LED effect



1 UDP Broadcast Discovery
---------
Discovery of Twinkly devices on the local network subnet.

1. Application sends UDP broadcast to port 5555 with message **\\x01discover** (first character is byte with hex representation 0x01).
2. Server responds back with following message:

   - first four bytes are octets of IP address written in reverse - first byte is last octet of the IP address, second second to last, ...

   - fifth and sixth byte forms string "OK"

   - rest is string representing `device name`_ padded with zero byte.

Example Send: 
01646973636f766572

Example Send Breakdown:
01 (.)
646973636f766572 (discovery)

Example Response:
0a02a8c04f4d7477696e6b6c795f42363932313600

Example Response BreakDown:
0a02a8c04f4d (ip 192.168.2.11) 
74 (OK)
77696e6b6c795f423131313131 (twinkly_B11111)
00 (padding)



2 http GET gestalt (get device info)
---------
Application uses http POST on port 80 to get device info

Example:
Host: 192.168.2.11
Port: 80
Method: GET
URL: /xled/v1/gestalt

Example Response (JSON Text)
{
	"product_name": "Twinkly",
	"product_version": "2",
	"hardware_version": "6",
	"flash_size": 16,
	"led_type": 5,
	"led_version": "1",
	"product_code": "TW105SEUM06",
	"device_name": "twinkly_B11111”,
	"uptime": "3978004",
	"hw_id": "00000000”,
	"mac": “00:00:00:00:00:00”,
	"uuid": "00000000-0000-0000-0000-000000000000",
	"max_supported_led": 224,
	"base_leds_number": 105,
	"number_of_led": 224,
	"led_profile": "RGB",
	"frame_rate": 14,
	"movie_capacity": 719,
	"copyright": "COMPANYNAME YEAR",
	"code": 1000
}

gestalt provide device hardware info to the client. 
product_version - 0 = 2016, 1 = 2017 ?
led_type - ?
uuid - does not seem to be used at this version of the protocol. 
base_leds_number - provides the number of LED build into the device.
max_supported_led - provides the max number of LED the device can use, including the base number of LED. 
number_of_led - provides the current user set value for the number of LED to use. 
led_profile - provides the type of LED coloring the device recognizes and uses. (‘RGB’ vs Special Edition ((not sure what value special edition may return)))
frame_rate - Current or maybe max frames rate of device?
movie_capacity - provides the movie capacity of the device. 
device_name - device label for the device, by default generated by code. But value can be set/changed by API. Basically user's desired name for device. Helpful if you have multiple sets of lights.
product_code - informs the client of the device product model. The model be be used to in oder for the client to know details about the device. Maybe this include lights vs a tree, wreath, cone, extra? maybe to tell what LED add-on strips can be offered? In the case of TW105SEUM06 maybe that means the clients knows it can set number_of_led to:  105 (base LED no add on LED strips), 161 (base 105 LED + 56 LED strip), 224 (base 105 LED + 119 LED strip) 



3 http POST login (generate authentication_token) 
---------
client uses http POST on port 80 to generate authentication_token

Example Send:
HOST: 192.168.2.11
Port: 80
Method: GET
URL: /xled/v1/login
JSON Text:
{
	"challenge": "v+ePUQ7uIpFVgVkauDjTTraWaN8Dg6oHFYAPLSoWNt8="
}

Example Response:
{
	"authentication_token": "vWUWUJYWpYA=",
	"authentication_token_expires_in": 14400,
	"challenge-response": "41680fb0b3c1a2fecf5494b5da0b7720aed2056b",
	"code": 1000
}

authentication_token the session token the client, authentication_token is require for for most commands sent to the server. The authentication_token is included in the header of http GET and POST.
authentication_token_expires_in how long the client session token is good for
challenge-response response to from server to client to identify server
code the servers error response, 1000 is successful, any other value seems to indicate error. 

The server will only respond to the last authentication_token created. 
Example: 
client A created token 1
client A verifies token with server, the response is code 1000 (successful)
client B creates token 2
token 1 now fails verification  & can not be used.

When a client request an authentication_token from the server, the client send a challenge, when the server response back with an authentication_token, the server includes a challenge-response. 

While there is not indication that the server verifies the challenge. In future versions of the protocol the server could review the challenge to verify a shared secret. In which case, the server might only return to the client an authentication_token if the correct  session sending the correct secret inside of the challange. 

Similarly while there is no indication the client currently verifies the challenge-response. The client could in theory verify the shared secret. In which case, the client might only only attempt to issue commands to servers that respond with the correct secret. 

The Server can chose to verify the challenge to determine if it wants to respond to the client.
The Client can chose to verify the response yo determine if it wants to talk to the server.

1. Generate encryption key

   1. Use secret key: **evenmoresecret!!**
   2. get byte representation of MAC address of a server and repeat it to length of the secret key
   3. xor these two values

2. Encrypt - use rc4 to encrypt challenge with the key

3. Generate hash digest - encrypted data with SHA1

4. Compare - hash digest must be same as challenge-response from server



4 Verification of challenge-response
---------

Example Send:
HOST: 192.168.2.11
Port: 80
Method: GET
URL: /xled/v1/verify
Header: X-Auth-Token: vWUWUJYWpYA=

JSON Text:
{
}

Example Response:
{
	"code": 1000
} 

1000  = success, other value would be error/failed



5 http GET status
---------
Example Send:
HOST: 192.168.2.11
Port: 80
Method: GET
URL: /xled/v1/network/status
Header: X-Auth-Token: vWUWUJYWpYA=

JSON Text:
{
}

Example Response:
{
	"mode": 1,
	"station": {
		"ssid": "Wireless Network Name",
		"bssid": "MAC ADDRESS",
		"ip": "Twinklys IP Address",
		"gw": "Router IP Address",
		"mask": "Wireless Subnet Mask",
		"status": 5
	},
	"ap": {
		"ssid": "Device Name",
		"channel": 1,
		"ip": "0.0.0.0",
		"enc": 0
	},
	"code": 1000
}

mode - 0 = mode ap? 1 = station?
status - might be 

mode 1 / station - seems to be what the device is set to use to connect to a wifi network?
mode 0 / ap - seems to be what the device would use to create it's own network, if the device is not connected to a network

for mode ap, ssid seems to default to device_name. I haven't tried changing the device name to see if that would change the ssid for ap


6 http GET firmware version
---------
Example Send:
HOST: 192.168.2.11
Port: 80
Method: POST
URL: /xled/v1/fw/vertion
Header: X-Auth-Token: vWUWUJYWpYA=

JSON Text:
{
}

Example Response:
{
	"version": "firmware version here",
	"code": 1000
}


7 http ?POST? update firmware
---------

Update sequence follows:

1. application sends first file to endpoint 0 over HTTP
2. server returns sha1sum of received file
3. application sends second file to endpoint 1 over HTTP
4. server returns sha1sum of received file
5. application calls update API with sha1sum of each stages.


8 LED effect operating modes
---------
Example Send:
HOST: 192.168.2.11
Port: 80
Method: POST
URL: /xled/v1/led/mode
Header: X-Auth-Token: vWUWUJYWpYA=

JSON Text:
{
	"mode": "rt"
}

Example Response:
{
	"code": 1000
}


Hardware can operate in one of following modes:

- off - turns off lights
- demo - starts predefined sequence of effects that are changed after few seconds
- movie - plays last uploaded effect
- rt - receive effect in real time


Mode off
----------------------------
1. Application calls API to switch mode to off

Device will set all LED to value of off. 

Mode demo
----------------------------
1. Application calls HTTP API to switch mode to demo

Device will play built in demo mode.
Not sure if this is a script doing on onboard version of RT, or if this is a built-in movie effect file.


Mode movie
----------------------------
1. Application calls HTTP API to switch mode to demo

Device will play the api set movie mode file currently stored on device. 


9 Upload full movie LED effect
----------------------------

1. Application calls HTTP API to switch mode to movie
2. Application calls API movie/full with file sent as part of the request
3. Application calls config movie call with additional parameters of the movie (such as frame_rate)

Movie file should not exceed capacity defined in device hardware as movie_capacity. 



Movie file format
-----------------

LED effect is called **movie**. It consists of **frames**. Each frame defines colour of each LED.

Movie file format is simple sequence of bytes. Three bytes in a row represent intensity of *red*, *green* and *blue* in this order. Each frame is defined just with number of LEDs times three. Frames don't have any separator. Definition of each frame starts from LED closer to LED driver/adapter.


9 mode rt
(Real time LED operating mode)
----------------------------

1. Application calls HTTP API to switch mode to rt
2. Then UDP packets are sent to a port 7777 of device. *Each packet represents single frame* that is immediately displayed. See bellow for format of the packets.
3. if no UDP packet is sent, after 60 seconds rt time out, and the device will revert to mode movie.


Real time LED UDP packet format
-------------------------------

Before packets are sent to a device application needs to login and verify authentication token. See above.

Each UDP has header:

* 1 byte *\\x01* (byte with hex representation 0x01)
* 8 bytes Base 64 decoded authentication token
* 1 byte number of LED definitions in the frame

Then follows body of the frame similarly to movie file format - three bytes for each LED.

For my 105 LED each packet is 325 bytes long.


Scan for WiFi networks
----------------------
When you firs setup twinkly, it creates it's own wifi network.
To get it to join your wifi network, you need to connect to twinkly's wifi network, and then tell twinkly the wifi network to join.
Twinkly's wifi card may not support the same wifi standards are your smartphone. As such, it scans.  
I assume there is also a command to set the wifi network to join.  

Hardware can be used to scan for available WiFi networks and return some information about them. I haven't seen this call done by the application so I guess it can be used to find available channels or so.

1. Call network scan API
2. Wait a little bit
3. Call network results API


On Error
--------------------------

At any time,
Response from POST or GET could change from 1000 to another code.
At that point API needs to perform Device Handshake to re-establish connection to device.