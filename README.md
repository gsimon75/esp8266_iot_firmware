# My custom ESP8266 IoT firmware

First of all, why another one?

- Tasmota is a great one. Tons of supported controllers, full support for custom boards even on pin-level,
  fully usable web ui, MQTT support. *The* best. (On my lightbulbs and switches I'm using Tasmota, btw.)

  But it's written in Arduino lingo and uses PlatformIO and its general scheme isn't designed to be
  realtime, and being easy to configure needed some hardwired assumptions, like you can't control more than one
  RGB PWM lightsources with it.
  
  Its primary goal is to support _smart devices_ that should do one thing only, and not to support
  _anything_ that one can wire up to an ESP8266 module on a homemade experimental board.

- NodeMCU and MicroPython are also great ones. Fully ~configurable~ scriptable, so one can implement
  anything with them. As long as it doesn't require down-to-the-microsecond timing and direct control
  of the hw resources. Great for the homemade franken-light-controllers, but not for realtime.
  (I've tried to do stepper motor control with them, it wasn't realtime enough to give a smooth run,
  especially not when the load-dependent timing would've become prevalent.)

- Espurna is also a great firmware, but it focuses on single-purpose stock devices even more than Tasmota.
  Its concepts are clean and the codebase is tidy and all that, but it's not for this particular goal either.

So, no matter how I try to avoid it, it seems I do need my own firmware after all. Not a big problem, the SDK is fine,
I just don't fancy the idea of reinventing wheels...


## The goals

1. Pure C, full control over the chip (OK, as much control as the SDK leaves me)
2. MQTT wherever possible, plain TCP socket otherwise (we'll see)
3. Concept: control graph - pins, controllers, parameters, connections
4. Store the config in flash, but don't flash any state data. Normal operations should't write the flash.
5. No UI, no localization, no predefined configs. Sorry, the user needs at least an MQTT client and do some copy-pasting from the docs


## Reset and low-level config

As of hw, we need *one* pin for "maintenance boot" mode and status led operations.

- output and driven high:
- output and driven low:
- input and reads high:
- input and reads low:

Configuring will be done via MQTT as well, but for that we need an MQTT connection, that is:
- Wifi SSID and password
- MQTT server ip, user, password

So, after flashing or "maintenance boot" the unit will:
- Set up wifi as AP (SSID generated from serial number, pw is some predefined nonce)
- Provide a minimal DHCP service, accept one connection and assign 192.168.1.1 to it
- Give feedback with led blinking
- Wait for client connection
- Give feedback with led blinking
- Start listening on 23/tcp (telnet)
- Process commands like 'wifi_ssid qwerqwer', 'wifi_pass qwerqwer', 'mqtt_ip 1.2.3.4', 'mqtt_user qwerqwer' and 'mqtt_pass qwerqwer'
- Acknowledge commands with 'OK'/'ERR'

From this on, all communication can go through via MQTT, with messages like `{cmnd,stat,cfg},{serial,name},...`

