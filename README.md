yaota8266
=========

howto
-----

Compile your `yaota8266.bin`:

```
.../yaota8266$ make build
```

The `build` target will do this for you:

* generate RSA keys, store them in: `.../yaota8266/ota_client/priv.key` and `.../yaota8266/ota_client/pub.key`
* Create `config.h` as a copy from `config.h.example`
* Update `#define MODULUS "..."` with the generates RSA keys

**Note:** You need to setup the complete toolchain. For easier usage via docker, go to: https://github.com/jedie/docker-yaota8266/ ;)


Existing helper files for ESP8266 device:

* `helpers/force_yaota8266.py` - Set OTA magic word in RTC RAM and reset device to force start OTA Server
* `helpers/verify_device.py` - Verify if yaota8266.bin flashed with the same RSA key on the device


Existing `make` targets:

```bash
.../yaota8266$ make
make targets:
  help              This help page
  update-config     Create config.h if not exists and/or insert current RSA modulus in config.h
  print-rsa-modulus Print the RSA modulus line for copy&paste into config.h
  rsa-keys          Generate RSA keys and print the RSA modulus line for copy&paste into config.h
  verify            Check RSA key, config.h and compiled "yaota8266.bin"
  build             Build boot8266 and ota-server and combine it to: "yaota8266.bin" and verfiy it
  clean             clean builded files
```


OTA-Client cli, e.g.:

```bash
.../yaota8266$ ./cli.py -h
usage: cli.py [-h]
              {update_config,generate_rsa_keys,print_rsa_modulus,sign,ota,canned_ota,verify}
              ...

yaota8266 (yet another esp8266 OTA) client

optional arguments:
  -h, --help            show this help message and exit

subcommands:
  {update_config,generate_rsa_keys,print_rsa_modulus,sign,ota,canned_ota,verify}
    update_config       Create config.h if not exists and/or insert current
                        RSA modulus in config.h
    generate_rsa_keys   Generate RSA keys in '.../yaota8266/ota_client/' if
                        not already exists
    print_rsa_modulus   Print the RSA modulus line for copy&paste into
                        config.h
    sign                Sign firmware file for OTA
    ota                 Do the OTA update for a device (need a non-signed firmware-ota.bin because this command sign the firmware before send it)
    canned_ota          Do the 'canned' OTA update for a device
    verify              Check RSA key, config.h and compiled 'yaota8266.bin'
```



background information
----------------------

yaota8266 is yet another bootloader/over-the-air (OTA) update solution
for ESP8266 WiFi SoC. Unlike many other solutions, yaota8266 does not
require reserving FlashROM space of 2x the size of the firmware. Instead,
it updates the firmware in-place. Of course, this means that if an OTA
update fails, there's no previous firmware to fallback to. On the other
hand, if OTA update fails, you likely will repeat it again, until it
succeeds. So, for many usecases the process of OTA update will be the
same - a user will just repeat it until it succeeds, regardless whether
there's a fallback firmware or not.

yaota8266 is written with big firmwares and small flash sizes in mind.
For example, it allows to have an OTA for full-fledged MicroPython
(firmware sizes of 512+KB) on 1MB flash devices, and still have a
small, but a filesystem.


Structure and algorithm
-----------------------

yaota8266 consists of two parts:

* 2nd-stage bootloader boot8266
* ota-server application

boot8266 works in the following way:

1. 1st-stage bootloader in ESP8266 BootROM loads boot8266 (from sector 0).
   It is small and fits within a single FlashROM sector (4K).
2. boot8266 checks whether an OTA button on device is pressed. If
   it is, it goes in OTA mode.
3. If the button is not pressed, it verifies a checksum of a user
   application. If it fails (for example, because of unsuccessful,
   partial previous firmware update), it goes into OTA mode.
4. If OTA mode is requested, boot8266 loads an application starting
   at the sector 1. This is intended to be the ota-server, but from
   boot8266's point of view, it's just a standard ESP8266 application,
   which it loads recursively in the same (or very similar) way as
   BootROM does it.
5. If OTA mode was not requested, boot8266 loads a user application
   which lies beyond the ota-server application end (offset is
   configurable). The same note as above applies - boot8266 just loads
   one or another application in the same way, and doesn't care what
   they do (but boot8266 has partially hardcoded knowledge about sizes
   of these applications, and verifies checksum only of the second one).

ota-server works in the following way:

1. Starts a UDP server on port 8266.
2. Expects consecutive UDP datagram containing chunks of new firmware.
3. Each datagram is signed with RSA private key. Only someone with
   a valid private key may produce valid datagrams, information from
   which ota-server will flash as a user application. (The public key
   is configured when building ota-server.)
4. ota-client host-side application is provided to drive OTA upgrade
   process for a device in OTA mode.

Known issues
------------

yaota8266 is a work in progress and is not yet fully working per the
spec above.
