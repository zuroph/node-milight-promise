# node-milight-promise

[![Build Status](https://travis-ci.org/mwittig/node-milight-promise.svg?branch=master)](https://travis-ci.org/mwittig/node-milight-promise)
[![Coverage Status](https://coveralls.io/repos/github/mwittig/node-milight-promise/badge.svg?branch=master)](https://coveralls.io/github/mwittig/node-milight-promise?branch=master)

A node module to control Milight LED bulbs and OEM equivalents such as Rocket LED, Limitless LED Applamp, 
 Easybulb, s`luce, iLight, iBulb, and Kreuzer. This library uses Promises to automatically synchronize the command 
 sequences. Thus, there is no need for nesting commands using callbacks. Of course, each API call returns a promise 
 which can be used to wait for the call to be resolved or rejected. The module has been tested with RGBW and White 
 bulbs using a Milight version 4 bridge. RGB bulbs which are no longer sold since January 2014 should also work using
 the rgb command set.

## Introduction

Milight uses a very primitive one-way communication protocol where each command must be sent in a 
 single UDP packet. It is just fire & forget similar to simple RF protocols for garage door openers and such.
 Compared to other Milight libraries, I am using a more aggressive timing for the delay between sending UDP command 
 packets (```delayBetweenCommands``` property). 
 
 Generally, the delay is to reduce the chances of UDP packet loss on the network. A longer delay may lower the risk of 
 data loss, however, data loss is likely to occur occasionally on a wireless network. Keep in mind, that apart from your 
 Wifi network there is another lossy communications channel between the Milight Controller and the bulbs. My strategy 
 against loss is to repeat each command. By default it will be send three times (```commandRepeat``` property). 
 
## What's new

Support for V6 bridge is developing :) 
For now, see the examples folder.

### V6 Feature summary

* support for full color (untested), RGBW, WW/CW, and bridge lights
* optional support for the legacy milight color wheel
* flow control with automatic retry
* control for multiple bridges
* bridge discovery
* support for unicast and broadcast modes

### TODO

* add missing bits and bobs to the command set like night mode and maximum brightness
* further improve flow control
* implement session lifetime
* support for keep-alive command messages
* explore use of password bytes
* test with full color bulb

### Breaking Changes

The old command interface should work as is with the following exception:

* the command function `rgbToHsv()` has ben removed to reduce the amount of code duplication. If you use this 
  function in your code you can now find the code in `helper.js` which is also exported using the `helper` property.
  
There are also some subtle changes of the default behaviour:

* the default for `commandRepeat`has been changed to `1` to provide consistent operation for all modes (WW/CW bulbs 
  with legacy bridges in particular). Note, beyond, a higher value does not make much sense for V6 as the flow control
  strategy will automatically resend messages as required (if no receipt received)
  
* the default for `delayBetweenCommands`has been changed to `100` ms to provide consistent operation for all modes. 
  According to my findings a more aggressive timing is possible with the legacy bridges depending on condition of the 
  LAN (quality of the wireless link and router physics).  

### Flow Control

The new bridge allows for the implementation of flow control as each command message received by the bridge is replied 
with a receipt message. So far, I have implemented a very simple strategy which is to wait for the receipt 

### Color Model

To my surprise, the color model of the V6 bridge is different to the color wheel supported by earlier 
bridge versions for RGBW lights. If you look at the old color wheel the new color wheel runs counterclockwise 
and it is shifted by about 22.5 degrees to the left. It still looks odd to me as it does not match with RGB 
wheel or the HSV color circle. Another curiosity is the color wheel of the bridge light is different to the 
one of the RGBW light as it is shifted by another 22.5 degrees (starting with red instead of violet). 
Maybe this is a bug of the bridge firmware or it is a calibration issue?

To cut a long story short I have integrated an optional transformation feature to the hue command for bridge and rgbw 
to optionally support the legacy color wheel. This might be useful for applications which provide controls
for the legacy color wheel like the plugin for pimatic does.

## Features

### Brightness

I noticed the `rgbw.brightness()` command never reached the maximum brightness level of the bulb and it turned out to be
yet another bug in the `commands` file. I also found an article suggesting the RGBW bulbs support 22 brightness levels 
instead of 20, however, the two additional levels did not change the brightness for me (tested with 6W bulbs). Maybe 
this is different with 9W bulbs. For this reason, I have added `rgbw.brightness2()` which maps brightness 0-100 to 
22 levels.

Another interesting observation is the RGBW bulbs keep brightness levels for color mode and white mode, individually.
Thus, you may notice a change in brightness if you switch to white mode, for example. So, if your application only
maintains a single brightness control you need to make sure to send the commands in the right order!

### Rendering RGB colors

For RGBW bulbs the command `rgbw.rgb255` is provided to map RGB values to Milight. However, the
color space of Milight is limited, as it is not possible to control color saturation and there are only 20 
brightness levels. Thus, the results may be disappointing when compared to other LED lightning technologies.
Effectively, Milight is unable to display different shades of grey.
  
### 2-byte Command Sequences

Recently, I found out that Milight bridge version 3 and higher can also handle 2-byte command sequences instead of
3-byte command sequence. Basically, the last byte of the 3-byte command sequence, call it stop byte, can be omitted. 
It is said the 2-byte command sequences provide better performance. This may be true for the Milight RF protocol, 
but I don't think it has an significant impact on the IP communication between the application and the bridge. To use 
the 2-byte command sequences you can simply use `commands2` instead of `commands`. Mixing 2-byte and 3-byte sequences 
is also supported.

### Bridge Discovery

The new bridge discovery function can be used to discover the IP and MAC addresses of Milight v4 Wifi bridges found 
on the local network. The following options can be provided to the discovery function.

| Property  | Default           | Type    | Description                                 |
|:----------|:------------------|:--------|:--------------------------------------------|
| address   | "255.255.255.255" | String  | The broadcast address                       |
| timeout   | 3000              | Integer | The timeout in milliseconds for discovery   |

An array of results is returned. Each result contains the following properties:
* ip: The IP address string
* max: The MAC address string

## Usage Example

See also example code provided in the `examples` directory of the package.

```javascript
var Milight = require('node-milight-promise').MilightController;
var commands = require('node-milight-promise').commands2;

// Important Notes:
// Instead of providing the global broadcast address which is the default, you should provide the IP address
// of the Milight Controller for unicast mode. Don't use the global broadcast address on Windows as this may give
// unexpected results. On Windows, global broadcast packets will only be routed via the first network adapter. If
// you want to use a broadcast address though, use a network-specific address, e.g. for `192.168.0.1/24` use
// `192.168.0.255`.

var light = new Milight({
		ip: "255.255.255.255",
		delayBetweenCommands: 75,
		commandRepeat: 2
	}),
	zone = 1;

light.sendCommands(commands.rgbw.on(zone), commands.rgbw.whiteMode(zone), commands.rgbw.brightness(100));
light.pause(1000);

light.sendCommands(commands.rgbw.off(zone));
light.pause(1000);

// Setting Hue
light.sendCommands(commands.rgbw.on(zone));
for (var x = 0; x < 256; x += 5) {
	light.sendCommands(commands.rgbw.hue(x));
	if (x === 0) {
		commands.rgbw.brightness(100)
	}
	light.pause(100);
}
light.pause(1000);

light.sendCommands(commands.rgbw.off(zone));
light.pause(1000);

// Back to white mode
light.sendCommands(commands.rgbw.on(zone), commands.rgbw.whiteMode(zone));
light.pause(1000);

// Setting Brightness
light.sendCommands(commands.rgbw.on(zone));
for (var x = 100; x >= 0; x -= 5) {
	light.sendCommands(commands.rgbw.brightness(x));
	light.pause(100);
}
light.pause(1000);

light.sendCommands(commands.rgbw.off(zone));
light.pause(1000);

light.close().then(function () {
	console.log("All command have been executed - closing Milight");
});
console.log("Invocation of asynchronous Milight commands done");
```

## Usage example for Discovery

```javascript
var discoverBridges = require('../src/index').discoverBridges;

discoverBridges().then(function (results) {
	console.log(results);
});
```
    
## Important Notes

* Instead of providing the global broadcast address which is the default, you should provide the IP address 
  of the Milight Controller for unicast mode. Don't use the global broadcast address on Windows as this may give
  unexpected results. On Windows, global broadcast packets will only be routed via the first network adapter. If
  you want to use a broadcast address though, use a network-specific address, e.g. for `192.168.0.1/24` use
  `192.168.0.255`.
* For White bulbs the property `commandRepeat` should be set to `1`, as the brightnessUp/brightnessDown, and
  warmer/cooler commands will perform multiple steps otherwise.
    
## History

See [Release History](https://github.com/mwittig/node-milight-promise/blob/master/HISTORY.md).

## License 

Copyright (c) 2015-2016, Marcus Wittig and contributors. All rights reserved.

[MIT License](https://github.com/mwittig/node-milight-promise/blob/master/LICENSE)
