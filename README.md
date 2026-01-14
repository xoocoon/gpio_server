# gpiosrv
An asyncio-based service translating GPIO signals into Linux kernel key events.

Developed and tested for Raspberry Pi variants, namely Pi 3B, 4B and Pico 1 attached via USB. On Pi 3B and Pi 4B, [pigpiod](https://abyz.me.uk/rpi/pigpio/pigpiod.html) is used as a backend for evaluating GPIO edges. As this does not work on Pi 5B any more, [libgpiod](https://libgpiod.readthedocs.io/en/latest/) is currently adapted as an alternative backend.

For the adoption of Raspberry Pico, the pre-existing [picod project](from https://abyz.me.uk/picod/index.html) is used. That is, only the picod C program is used from that project, whereas the client-side Python module was completely re-written and heavily extended in the scope of *gpiosrv*. Eventually, the GPIOs of a Raspberry Pico, attached to a Linux machine via USB 2.0 can be controlled largely as if they were native GPIOs on the Linux host machine.

For a minimal setup, one central instance of `gpio_server.py` running on a Linux machine is required. Its main task is to translate pre-defined signals received on corresponding GPIOs into key events. Additionally, a server instance can be accessed from CLI tools via a Unix socket. In a more advanced setup, multiple instances of `gpio_server.py` may run on one Linux machine. In this case, each instance is exclusively attached to one available GPIO chip. For example, on a Raspberry Pi 4B, this might be the native GPIO chip of the 40 pin header, and additionaly a Pico 1 attached via USB 2.0. Both instances can then be used by clients in the same manner, as long as the desired target is addressed properly.

Since *gpiosrv* is based on asyncio in Python, and asyncio in turn is backed by a C implementation in most of the Python interpreters, the processing runs fast enough for most use cases, even on single board computers.

## Signal definition language

Various signal transmitters can be used with a `gpio_server.py` instance. For translating pulsed and other signals into key events, a `gpio_server.py` instance requires protocol definitions in one or more JSON files. The following is an example for capturing a pulsating door bell signal. On the hardware side, whenever the door bell rings, a rectifier and Z diode emits pulses to GPIO 25. With the following JSON declarations, a `KEY_SOUND` key press is emulated.

```
{
  "evdevName": "pi-sig-injector",
  "gpios": {
    "25": "door-bell"
  },
  "signalProtocols": {
    "gate-bell": {
      "basePulse_μs": 9000,
      "lengthTarget_basePulseCount": 5,
      "codeStartBit": 1,
      "preamble_ms": 10,
      "postamble_ms": 12,
      "tolerancePercentage": 30,
      "debounce_μs": 600,
      "signalListener": {
        "mayInject": false
      },
      "keys": {
        "0x1f": {
          "keyName": "KEY_SOUND"
        }
      }
    }
  }
}
```

These declarations state that, whenever at least 5 pulses with a length of 9000 microseconds each, are received on GPIO 25 with a timing tolerance of plus/minus 30%, a pair of a key down and a key up event for `KEY_SOUND` shall be propagated via the Linux kernel. Of course, such emulated "key presses" can be processed by another instance on the same Linux machine, triggering virtually any action. 

To this end, the repo features an additional service. It may be deployed in one or more instances, each grabbing and processing an arbitrary subset of  key events. Unless such a processing service is deployed, the effect is no different to really pressing a key on a keyboard. This opens up a versatile approach for decoupling signal transmitters and receivers in an ecosystem of Linux deamons, optionally orchestrated by systemd.

Another signal source might be an IR remote control. In this case, the processing service can be omitted entirely when there is already a media center software processing the translated key presses. The following is an example for capturing the pulses of an MCE-compatible remote control.

```
{
  "evdevName": "pi-ir-injector",
  "gpios": {
    "19": "Media Center Edition"
  },
  "irProtocols": {
    "Media Center Edition": {
      "baseProtocol": "RC6_MCE",
      "address": "0x800f",
      "irListener": {
        "mayInject": true
      },
      "keys": {
        "0x0411": {
          "keyName": "KEY_VOLUMEDOWN"
        },
        "0x0410": {
          "keyName": "KEY_VOLUMEUP"
        }
      }
    }
  }
}
```

By contrast to the previous example, the configuration above relates to a specific IR protocol handler instead of a generic signal handler. This demonstrates that *gpiosrv* can be extended by individual protocol handlers and corresponding JSON configurations.

## Key definitions

The key definitions used by *gpiosrv* correspond to the constants used by the Linux kernel, as defined in [input-event-codes.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h).

In the examples above, these are `KEY_SOUND` and `KEY_VOLUMEUP` / `KEY_VOLUMEDOWN`, respectively.

## Key processing language

As stated above, the repo offers an additional service processing the key events generated by a central `gpio_server.py` instance. It executes pre-defined commands on the Linux machine whenever a specific key event is triggered. The following is an example for commands to execute whenever the button F12 is pressed (`shortPressCommands`) or held down (`longPressCommands`).

```
"keys": {
    "KEY_F12": {
      "shortPressCommands": [
        [
          "/usr/bin/curl",
          "--netrc-file",
          "/mnt/encrypted/rest.netrc",
          "https://mediabox.local:6863/standby"
        ]
      ],
      "longPressCommands": [
        [
          "/usr/bin/curl",
          "--netrc-file",
          "/mnt/encrypted/rest.netrc",
          "https://mediabox.local:6863/shutdown"
        ]
      ],
      "ledColor": "orange"
    }
}
```

The example also shows that an indicator LED can be triggered with a specific color upon key events. If you come to the conclusion that a key processing service can be used independently of a `gpio_server.py` instance, you are right. Imagine a little keypad attached to your single board computer. As the keypad generates already genuine key events in the Linux kernel, you do not need a `gpio_server.py` instance in the first place. It is sufficient to set up a key processing service with a configuration file like the one above in order to execute pre-defined commands.

# Current state

The code for the functionality described above already exists and has matured over years. Hopefully, I will find the time to curate and document the code up to a state eligible for its sharing as OSS.

Please feel free to contact me if the functionality might be useful to you. I might share portions of the code beforehand on a bilateral basis.
