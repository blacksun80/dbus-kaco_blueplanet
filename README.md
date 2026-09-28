# dbus-kaco_blueplanet Service
Victron Venus integration for Kaco blueplanet 3.0 TL3 - 10 TL3 Inverters

### Purpose

This service is meant to be run on a raspberry Pi with Venus OS from Victron or a for example a Cerbo GX device.

The Python script cyclically reads data from the Kaco blueplanet Inverter via Sunspec Modbus and publishes information on the dbus.

By default, only two services are actually registered: com.victronenergy.pvinverter.pv0 (PV inverter power/energy) and com.victronenergy.temperature (inverter cabinet temperature). The code also contains a com.victronenergy.grid service (would make Venus OS work as if a physical Victron Grid Meter were installed) and a com.victronenergy.digitalinput service (limit-mode indicator), but both are commented out in `new_service()` near the bottom of the script and are not created unless you uncomment them yourself.

**Note:** if all you need is the plain PV inverter reading (AC power/voltage/current, energy, status/error code) and your inverter's own SunSpec Modbus TCP interface is reachable directly (not just through a gateway that remaps the register addresses), check first whether Venus OS's built-in generic SunSpec PV inverter support (the same driver used for Fronius, `dbus-fronius`) already detects your inverter under *Settings → PV inverters* before setting up this script — it may work out of the box and needs no maintenance across Venus OS updates. It has been confirmed to work directly with a Kaco blueplanet 8.6 TL3 (firmware V5.53) reachable on its own IP; it only requires that a single Modbus TCP client can connect at a time, so a gateway/proxy that already holds the inverter's one allowed connection (as used in the setup below) will block it.

![Dashboard shows Energy flow](images/dashboard.png?raw=true "Dashboard")
![Menu shows Entries of the Inverter](images/menu.png?raw=true "Menu")

### Configuration

You need to modify the settings in the dbus-kaco_blueplanet.py as needed:

`SERVER_HOST = "192.168.178.80"`

`SERVER_PORT = 502`

`UNIT = 2`

### Installation

1. You need root access to your GX Device (https://www.victronenergy.com/live/ccgx:root_access)

2. Copy the files to the /data folder on your venus:

   - /data/dbus-kaco_blueplanet/dbus-kaco_blueplanet.py
   - /data/dbus-kaco_blueplanet/kill_me.sh
   - /data/dbus-kaco_blueplanet/service/run
   - /data/dbus-kaco_blueplanet/service/log/run

3. Set permissions for files:

  `chmod 755 /data/dbus-kaco_blueplanet/service/run`
  
  `chmod 755 /data/dbus-kaco_blueplanet/service/log/run`
  
  `chmod 744 /data/dbus-kaco_blueplanet/kill_me.sh`

4. Add a symlink to for auto starting:

   `ln -s /data/dbus-kaco_blueplanet/service/ /opt/victronenergy/service/dbus-kaco_blueplanet`

   The supervisor should automatically start this service within seconds, if not simply reboot your system.

### Upgrading Venus OS

If you are upgrading your Venus OS you will have to re-add the symlink for autostarting the python script (Repeat step 4 from the above installation instructions).

To avoid having to manually do this, create `/data/rc.local` with the following content (and `chmod 755 /data/rc.local`):

```sh
#!/bin/sh
if [ ! -e /opt/victronenergy/service/dbus-kaco_blueplanet ]; then
    ln -s /data/dbus-kaco_blueplanet/service/ /opt/victronenergy/service/dbus-kaco_blueplanet
fi
```

This gets executed late at every startup by `/etc/init.d/custom-rc-late.sh` (`/data/rc.local` is called directly, not via `sh /data/rc.local`).

**Important:** the shebang must be `#!/bin/sh` (not `!#/bin/bash`, a typo in an earlier version of this README) and the file must use Unix line endings (LF), not Windows (CRLF). Either mistake makes the kernel reject the script silently (`ENOEXEC`), so it never runs at boot — the symlink then stays missing after every Venus OS update until you reboot manually and it's basically the same as not having the file at all. Also wrap the `ln -s` in the existence check above so the script doesn't error out on runs where the symlink is already there.

### Debugging

You can check the status of the service with svstat:

`svstat /service/dbus-kaco_blueplanet`

It will show something like this:

`/service/dbus-kaco_blueplanet: up (pid 8179) 746 seconds`

If the number of seconds is always 0 or 1 or any other small number, it means that the service crashes and gets restarted all the time.

You could also take a look at the log-file:

`tail -f /var/log/dbus-kaco_blueplanet/current`

and see if there are any error messages.

When you think that the script crashes, start it directly from the command line:

`python /data/dbus-kaco_blueplanet/dbus-kaco_blueplanet.py`

and see if it throws any error messages.

If the script stops with the message

`dbus.exceptions.NameExistsException: Bus name already exists: com.victronenergy.grid"`

it means that the service is still running or another service is using that bus name.

If you see something like:

`2022-06-05 10:39:04,238 - DbusKaco - INFO - Startup, trying connection to Modbus-Server: ModbusTCP 192.168.178.80:502, UNIT 2`

`2022-06-05 10:39:04,247 - pymodbus.client.sync - ERROR - Connection to (192.168.178.80, 502) failed: [Errno 111] Connection refused`

`2022-06-05 10:39:04,249 - DbusKaco - ERROR - unable to connect to 192.168.178.80:502`

Then you are not able to connect to your Inverter via Modbus. This can be a misconfiguration or another client is already connected. 
The inverter will accept only one concurrent client connected, if you need more than one client connection you may use a modbus proxy like
https://pypi.org/project/modbus-proxy/


#### Restart the script

If you want to restart the script, for example after changing it, just run the following command:

`/data/dbus-kaco_blueplanet/kill_me.sh`

The supervisor will restart the scriptwithin a few seconds.

### Hardware

In my installation at home, I am using the following Hardware:

- Kaco blueplanet 8.6 TL 3
- Kaco blueplanet 10.0 TL 3
- 1x Victron MultiPlus-II - Battery Inverter (one phase)
- Cerbo GX (tested Firmware version: v2.87 and v2.92)
- DIY Battery 16x 280AH Lifepo EVE Cells with BMS from Batrium 

### Credits

I have shamelessly copied and adapted the code from https://github.com/h4ckst0ck/dbus-solaredge/blob/main/dbus-solaredge.py and the readme.
