# keyble

*keyble* provides an API and a set of command line tools for controlling/interfacing with [eQ-3 eqiva Bluetooth Smart Lock](https://www.eq-3.com/products/homematic/detail/bluetooth-smart-lock.html)s.

Available at prices as low as 60€, these Smart Locks are considerably cheaper than comparable smart locks. That doesn't mean they're lower quality or not as secure though - in fact, the eqiva Bluetooth Smart Lock came out first in a [comparative test by the highly-respected german "*Stiftung Warentest*" magazine](https://www.test.de/Smarte-Tuerschloesser-im-Test-Nicht-alle-sind-sicher-5655666-0/).

The device has one major downside though: The only intended way to control the smart lock is via the vendor's official Android and iOS smartphone app, making it an isolated application.
This is where *keyble* steps in. *keyble* allows controlling the smart lock from any computer that meets [the requirements](#requirements), making it possible to integrate the smart lock into existing smart home systems (based for example on [Node-RED](https://nodered.org/), [ioBroker](https://www.iobroker.net/), [Home Assistant](https://www.home-assistant.io/), [openHAB](https://www.openhab.org/), [FHEM](https://fhem.de/) etc.).

If you intend to use *keyble* to control an eqiva Bluetooth Smart Lock, you might also be interested in the cheap [eQ-3 eqiva Bluetooth Smart Radiator Thermostat](https://www.eq-3.com/products/homematic/detail/bluetooth-smart-radiator-thermostat.html)s from the same vendor. I wrote [a similar tool for controlling eQ-3 eqiva Bluetooth radiator thermostats](https://github.com/oyooyo/nixfilter-rtble) that works almost identical to *keyble*.

## *keyble* soon to be divided into *keyble* and *keyble-cli*

*keyble* will soon be divided into two separate npm packages, *keyble* and *keyble-cli*. *keyble* will then only contain the core library, to be used by other Node.js packages. The command-line scripts will move to *keyble-cli*. So if you're currently using *keyble*, you will then need to install *keyble-cli* instead.

## Current status

*keyble* currently only provides a few basic features:

    - Opening / locking / unlocking the smart lock
    - Registering new users
    - Toggling between locked and unlocked states

## ESP32 port

There is now also a [port of *keyble* for the ESP32 microcontroller](https://github.com/lumokitho/esp32-keyble) written in C++ that you may consider as an alternative. While I haven't tested it myself yet, it sounds very interesting! Basically, the idea is that you place a (wall socket-powered) ESP32 microcontroller board near the smart lock that exclusively acts like a kind of MQTT-bridge for the smart lock. So this approach requires a MQTT server running on your network, but I assume that most people interested in *keyble* use a MQTT server anyway.

One major hurdle the developers of the ESP32 port were facing was using BLE and WiFi at the same time. The workaround they found was that when a command to be sent to the smart lock is received via MQTT, the ESP32 will disconnect from MQTT and WiFi, turn WiFi off and BLE on, connect to the smart lock and send the command, then disconnect from the smart lock, turn BLE off and WiFi on and connect to MQTT again in order to wait for the next command. A minor disadvantage of this workaround is that there cannot be a permanent connection to the smart lock, so there will always be an unavoidable 1-2 second delay before the command will be executed because the connection has to be established first. But that is the default behaviour in *keyble* as well (unless you use the `--auto_disconnect_time 0` parameter) so for most people this probably doesn't even make a difference.

A huge advantage of the ESP32 port on the other hand is that it has exclusive access to the BLE hardware, and does not have to rely on a Node.js BLE library that is in a bit of an abandoned state since 2018, which I believe is the root of many problems people may be facing when using *keyble*.

As I understand, the original developer of the ESP32 port was [@MariusSchiffer](https://github.com/MariusSchiffer/esp32-keyble), with [@tc-maxx](https://github.com/tc-maxx/esp32-keyble), RoP09 and [lumokitho](https://github.com/lumokitho/esp32-keyble) greatly improving his work. *(If I forgot to mention someone, please let me know)*

## Requirements

*keyble* requires the following hard- and software:

- Bluetooth 4.0 compatible hardware
- [Node.js](https://nodejs.org/) *(version 10 or greater)*
- Linux, OSX or Windows operating system

## Installation

### Global installation

By installing *keyble* globally, *keyble*'s command line tools are installed in your PATH, and are therefor available from everywhere.

With Node.js/npm installed, you can install/update *keyble* globally by running on a command line:

    npm install --update --global --unsafe-perm keyble

The [`--unsafe-perm`](https://docs.npmjs.com/misc/config#unsafe-perm) flag seems to be necessary in order to install *keyble* globally via the `--global` flag (at least under Linux). If installing locally, without the `--global` flag, it works fine without the `--unsafe-perm` flag. This issue seems to be caused by one of *keyble*'s dependencies (see [#707](https://github.com/noble/noble/issues/707)).
You will probably need to run the above command with *sudo*, at least if using Linux.

*keyble* relies on a Node.js Bluetooth library called [*noble*](https://github.com/noble/noble/). If you have any problems installing/running *keyble*, chances are they are related to *noble* - therefor, it is generally advisable to read [the documentation on installing *noble*](https://github.com/noble/noble#prerequisites) if you witness any problems installing *keyble*.

In particular, please read [these remarks about *"Running without root/sudo"*](https://github.com/noble/noble#running-on-linux) if running on Linux.

### Local installation

To install/update *keyble* as a library/dependency instead, execute:

    npm install --update --save keyble

### Complete, step-by-step installation instructions for Debian-based Linuxes, especially for Raspberry Pi running Raspbian

The Raspberry Pi is probably the most popular platform to run *keyble* on, so I decided to provide complete, step-by-step installation instructions for that platform.
These instructions should, however, work on all other Debian-based Linuxes (like Ubuntu) as well.

    # (Optional, but recommended) Fully update/upgrade system
    sudo apt-get -y update && sudo apt-get -y dist-upgrade
    
    # Install Node.js LTS
    curl -sL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
    sudo apt-get install --upgrade -y build-essential nodejs
    
    # Make sure required libraries for Bluetooth are installed
    sudo apt-get -y install bluetooth bluez libbluetooth-dev libudev-dev
    
    # Install keyble
    sudo npm install --update --global --unsafe-perm keyble
    
    # (Optional, but recommended) Allow keyble to be run without sudo
    sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
    
    # (Optional, but recommended) Install tools for controlling via MQTT
    sudo apt-get -y install mosquitto-clients

## Command line tools

### keyble-registeruser

In order to actually control an *eQ-3 eqiva Bluetooth Smart Lock*, a user ID and the corresponding 128-bit user key is required.
Since the original app provides no way to get these informations, it is necessary to first register a new user, using the information encoded in the QR-Code of the "*Key Card*"s that ship with the lock.

This is what the *keyble-registeruser* tool is for.

    usage: keyble-registeruser [-h] [--user_name USER_NAME]
                               [--qr_code_data QR_CODE_DATA]
    
    
    Register users on eQ-3 eqiva Bluetooth smart locks.
    
    Optional arguments:
      -h, --help            Show this help message and exit.
      --user_name USER_NAME, -n USER_NAME
                            The name of the user to register (default: "PC")
      --qr_code_data QR_CODE_DATA, -q QR_CODE_DATA
                            The information encoded in the QR-Code of the key 
                            card. If not provided on the command line, the data 
                            will be read as input lines from STDIN instead

Usage example:

    keyble-registeruser -n John -q M0123456789ABK0123456789ABCDEF0123456789ABCDEFNEQ1234567
    
    Press and hold "Unlock" button until the yellow light flashes in order to enter pairing mode
    Registering user on Smart Lock with address "01:23:56:67:89:ab", card key "0123456789abcdef0123456789abcdef" and serial "NEQ1234567"...
    User registered. Use arguments: --address 01:23:56:67:89:ab --user_id 1 --user_key ca78ad9b96131414359e5e7cecfd7f9e
    Setting user name to "John"...
    User name changed, finished registering user.

#### Piping data into keyble-registeruser

If the QR-Code data is not passed on the command line via the `--qr_code_data/-q` argument, *keyble-registeruser* will read the data from STDIN instead. This allows simply piping the output of a QR-Code-Reader into *keyble-registeruser*.

For example, if you have a Webcam and the *[zbar](http://zbar.sourceforge.net/)* tools installed *(`sudo apt-get install zbar-tools`)*, you can run:

    zbarcam --raw | keyble-registeruser
    
    Press and hold "Unlock" button until the yellow light flashes in order to enter pairing mode
    Registering user on Smart Lock with address "01:23:56:67:89:ab", card key "0123456789abcdef0123456789abcdef" and serial "NEQ1234567"...
    User registered. Use arguments: --address 01:23:56:67:89:ab --user_id 1 --user_key ca78ad9b96131414359e5e7cecfd7f9e
    Setting user name to "PC"...
    User name changed, finished registering user.

The above command is the recommended way to register a new user under Linux.

### keyble-sendcommand

With a valid user ID and user key, as obtained by running the *keyble-registeruser* tool, we can now actually control (=open/lock/unlock/toggle) the Smart Lock.

This is what the *keyble-sendcommand* tool is for.

    usage: keyble-sendcommand [-h] --address ADDRESS --user_id USER_ID --user_key
                              USER_KEY [--auto_disconnect_time AUTO_DISCONNECT_TIME]
                              [--status_update_time STATUS_UPDATE_TIME]
                              [--timeout TIMEOUT]
                              [--command {lock,open,unlock,status,toggle}]
                          
    
    Control (lock/unlock/open/toggle) an eQ-3 eqiva Bluetooth smart lock.
    
    Optional arguments:
      -h, --help            Show this help message and exit.
      --address ADDRESS, -a ADDRESS
                            The smart lock's MAC address
      --user_id USER_ID, -u USER_ID
                            The user ID
      --user_key USER_KEY, -k USER_KEY
                            The user key
      --auto_disconnect_time AUTO_DISCONNECT_TIME, -adt AUTO_DISCONNECT_TIME
                            The auto-disconnect time. If connected to the lock, 
                            the connection will be automatically disconnected 
                            after this many seconds of inactivity, in order to 
                            save battery. A value of 0 will deactivate 
                            auto-disconnect (default: 30)
      --status_update_time STATUS_UPDATE_TIME, -sut STATUS_UPDATE_TIME
                            The status update time. If no status information has 
                            been received for this many seconds, automatically 
                            connect to the lock and query the status. A value of 
                            0 will deactivate status updates (default: 600)
      --timeout TIMEOUT, -t TIMEOUT
                            The timeout time. Commands must finish within this 
                            many seconds, otherwise there is an error. A value of 
                            0 will deactivate timeouts (default: 40)
      --command {lock,open,unlock,status,toggle}, -c {lock,open,unlock,status,toggle}
                            The command to perform. If not provided on the 
                            command line, the command(s) will be read as input 
                            lines from STDIN instead

Usage example:

    keyble-sendcommand --address 01:23:56:67:89:ab --user_id 1 --user_key ca78ad9b96131414359e5e7cecfd7f9e --command open

    MOVING
    OPEN
    UNLOCKED

#### Piping data into keyble-sendcommand

If the actual command/action ("open"/"lock"/"unlock"/"status"/"toggle") is not passed on the command line via the --command/-c argument, *keyble-sendcommand* will read the command(s) from STDIN instead. This allows piping the output of another program into *keyble-sendcommand*.

For example, if you have the *mosquitto-clients* tools installed *(`sudo apt-get install mosquitto-clients`)*, you could easily make your Smart Lock controllable via MQTT by running a command similar to this:

    mosquitto_sub -h 192.168.0.2 -t "door_lock/action" | keyble-sendcommand -a 01:23:56:67:89:ab -u 1 -k ca78ad9b96131414359e5e7cecfd7f9e | mosquitto_pub -h 192.168.0.2 -l -r -t "door_lock/status"

Assuming a MQTT broker with IP address 192.168.0.2, sending message "open" to the MQTT topic "door_lock/action" for example would then open the Smart Lock; changes to the door lock status would be automatically published as retained messages to MQTT topic "door_lock/status".

#### Installing as a mqtt-service
A scenario may be to run keyble on a single board computer (e.g. Raspberry Pi Zero). For this, it is needed to start the above mosquitto -> keyble-sendcommand -> mosquitto chain at boot.
For this, two files are needed:
* `/etc/systemd/system/door_lock.service`  - The systemd service
```
[Unit]
Description=Door lock
Require=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/door_lock.sh
Restart=always
StandardInput=tty
StandardOutput=tty
TTYPath=/dev/tty11

[Install]
WantedBy=multi-user.target
```

* `/usr/local/bin/door_lock.sh`  - The script that is run by the service.
```
#!/bin/bash
mqtt_server="192.168.177.3"
address="00:1a:22:12:12:12"
key="f98623423423442342342344322"
user_id="3"

/usr/bin/mosquitto_sub -h $mqtt_server -t "door_lock/action" | /usr/local/bin/keyble-sendcommand  --address $address --user_id $user_id --user_key $key  | /usr/bin/mosquitto_pub -h $mqtt_server -l -r -t "door_lock/status"
```

Both files are available in this github (under ./scripts/). The first file does not need to be edited. In the second file (the script), the mqtt-server ip, the address of the lock, the key and the user_id need to be configured.

*To install the service:*
* Download both files and put them at the correct location
* Edit the script with your credentials
* Make sure that the script is executable (`chmod u+x /usr/local/bin/door_lock.sh`)
* Start the service:
`sudo systemctl start door_lock`

* Enable it to run at boot:
`sudo systemctl enable door_lock`

## Troubleshooting and known issues

### *keyble* cannot work while the smartphone app is running

The smart lock can not handle two or more concurrent Bluetooth connections. So while the smartphone app is connected to the smart lock, *keyble* will not be able to connect to (and control) the smart lock as well. Since the smartphone app automatically connects to the smart lock and stays connected while it is running, *keyble* will not work while the smartphone app is running. So if you witness problems with *keyble* not working properly, please ensure that the smartphone app is closed.

### When sending a command to the smart lock via `keyble-sendcommand --command open`, it takes some 5-10 seconds until the smartlock actually executes the command. When I click the "open" button in the Smartphone app, it happens instantly

There are two main causes of this delay:

1. When running `keyble-sendcommand`, it takes some time before the program code will actually start executing. That's because the program code will first be JIT-compiled, required Node.js modules will first be loaded etc. How long this takes depends heavily on the hardware that keyble-sendcommand is running on; on my desktop PC, this takes around 1 second or so, on my Raspberry Pi, it takes some 4 to 5 seconds I'd guess.
The only solution to avoid this delay is to run keyble-sendcommand in continuous mode mode, without the "--command" option, as described in the "Piping data into keyble-sendcommand"-section. That way, keyble-sendcommand does not need to be initialized every time a command is being sent.

2. keyble-sendcommand needs to connect to the smart lock via Bluetooth LE and discover it's Bluetooth services and characteristics before it can actually send commands. That process takes some 1-1.5 seconds. This delay can be avoided by running `keyble-sendcommand` with the `-adt 0` command line option, which causes keyble-sendcommand to keep connected to the smartlock.
Be aware that `-adt 0` command line option only works in continuous mode, not when when sending a single command by executing `keyble-sendcommand --command ...`.

Disabling auto-disconnect by specifying `-adt 0` comes with 1-2 disadvantages though:
Since the PC running keyble-sendcommand will stay connected to the smart lock, no one else can connect to the smart lock anymore. So you could no longer control the smart lock via the Smartphone app as well for example. Furthermore, staying connected to the smart lock might drain the smart lock's batteries faster (but not necessarily - it might even be just the other way around, I've never really tried).

## API

Beware that since *keyble* is still in early alpha state, the API is likely to still change a lot, probably with backwards-incompatible changes. Only a subset of the functionality has been documented yet, and only a few usage examples are provided.

### Creating a *Key_Ble* instance

    // Require the keyble module
    var keyble = require("keyble");

    // Create a new Key_Ble instance that represents one specific door lock
    var key_ble = new keyble.Key_Ble({
        address: "01:23:45:67:89:ab", // The bluetooth MAC address of the door lock
        user_id: 1, // The user ID
        user_key: "0123456789abcdef0123456789abcdef", // The user-specific 128 bit AES key
        auto_disconnect_time: 15, // After how many seconds of inactivity to auto-disconnect from the device (0 to disable)
        status_update_time: 600 // Automatically check for status after this many seconds without status updates (0 to disable)
    });
    
### Lock / Unlock / Open / Toggle the door lock

    // Lock the door
    key_ble.lock()
    .then( () => {
        console.log("Door locked");
    });

    // Unlock the door
    key_ble.unlock()
    .then( () => {
        console.log("Door unlocked");
    });

    // Open the door
    key_ble.open()
    .then( () => {
        console.log("Door opened");
    });

    // Toggle the smart lock between locked and unlocked
    key_ble.toggle()
    .then( () => {
        console.log("Door state changed");
    });

### Listen for status changes

    // Lock the door
    key_ble.on("status_change", (new_status_id) => {
        console.log("New status:", new_status_id);
    });

### Timeouts for actions

*keyble* currently does not allow passing timeout values when calling actions like `key_ble.open()` which return [*Javascript Promises*](https://developers.google.com/web/fundamentals/primers/promises). As a result, the returned promise will often stay in *pending* state indefinitely if there is a problem, for example if the device is not in range. As a kind of compromise, there is a small helper function `keyble.utils.time_limit` which instead allows settings timeouts for every *Promise*:

    keyble.utils.time_limit(<promise>, <timeout_milliseconds>[, <timeout_error_message>])

It basically works like this: Instead of something like

    key_ble.open()
    .then( () => {
        console.log("Door opened");
    });

write

    keyble.utils.time_limit(key_ble.open(), 15000)
    .then( () => {
        console.log("Door opened");
    })
    .catch( (error) => {
        console.error("Error opening door!");
    });

to time-limit the open() action to 15000 milliseconds = 15 seconds.

## Beware of firmware updates

Be aware that the vendor might theoretically render this software useless with a future firmware update.

This software was developed against firmware version 1.7, which is the latest firmware version as of now *(2018/09/05)*.

If the vendor releases a newer firmware version, better not instantly update the firmware; wait for confirmation that the new firmware version is safe.

## ToDo

- Document API
- Switch to semantic versioning starting with version 1.0.0
- Create keyble-cli (CLI tools)
	- Add tool to configure lock to keyble-cli
	- Add tool to parse key card data?
	- Create a single executable
- Create keyble-mqtt (MQTT)
- Create keyble-nodered (Custom keyble node for Node Red)
- Create/Update/fix keyble-iobroker (Adapter for ioBroker)
- Hide sensible informations in output
- Convert to typescript?

## Acknowledgements

A big thanks to everyone who helped developing and improving this software.

Especially...

- [henfri](https://github.com/henfri), who provided lots of useful feedback and helped improving the code
- [Ircama](https://github.com/Ircama), who also provided lots of useful feedback and provided the toggle() command
