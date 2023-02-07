# hass-systemd v0.1.3

----UNDER DEVELOPMENT----

Allows Home Assistant to run as a systemd notify daemon, with watchdog support.

(C) Timothy Brown 2018.11.18    
(C) Frits  van Latum 2023.02.07

## Instructions

### Raspberry Pi

These instructions assume you're working with a Raspberry Pi, with **Raspberry PI OS** and a user **pi**
for logging in.
See [Raspberry Pi setup](#Raspberry%20Pi%20setup) for some tips about setting up your Raspberry Pi.

### Home Assistant

These instructions further assume you have installed Home Assistant Core following
the documentation in [Install Home Assistant Core](https://www.home-assistant.io/installation/linux#install-home-assistant-core).

The complete Home Assistant [installation documentation](https://www.home-assistant.io/installation/linux) on
linux describes four methods: (1) installation of Home Assistant with its own special operating system, (2) installation 
in a container, (3) plain installation on linux of Home Assistant Core and (4) the installation of Home Assistant Supervised.

So we are assuming the third method. This means:

- you have created a user **homeassistant**
- this user has **\home\homeassistant** as its home directory
- you have created a the directory **\srv\homeassistant** 
- and installed the Home Assistant software in a Python3 virtual environment.

During the installation a configuration directory **/home/homeassistant/.homeassistant** is created.

After installation Home Assistant is started by giving the next four commands:

`sudo -u homeassistant -H -s`

`cd /srv/homeassistant`

`source bin/activate`

`hass`

meaning: start a new session logged in as **homeassistant**, goto the installation directory, activate the
virtual environment and start Home Assistant with **hass**.

### Running Home Assistant automatically, as a deamon

Now we want to run Home Assistant automatically, without a terminal. It should also start automatically when
the Raspberry Pi is started. More in detail: we want Home Assistant to run as a systemd notify daemon, with watchdog support.

The linux operating system of your Raspberry Pi must have systemd. 

These are the instructions:
 

Make sure we're in our login user's home directory:

`cd ~`

Create the custom components directory, if needed:

`sudo -u homeassistant mkdir -p /home/homeassistant/.homeassistant/custom_components`


Download the plugin files:

`sudo git clone https://github.com/FritsvanLatum/hass-systemd.git /home/homeassistant/.homeassistant/custom_components/systemd`


Set file permissions:

`sudo chown -R homeassistant:homeassistant /home/homeassistant/.homeassistant/custom_components/systemd`


Edit your HA configuration to enable our new component:

`sudo -u homeassistant nano /home/homeassistant/.homeassistant/configuration.yaml`


and add the following line somewhere in the file:
    
`systemd:`


Edit the systemd service file to reflect your configuration:

`sudo -u homeassistant nano /home/homeassistant/.homeassistant/custom_components/systemd/hass.service`

(The `ExecStart=` and `User=` lines are the only ones that might be changed.)


Setup the systemd service:

`sudo chown root:root /home/homeassistant/.homeassistant/custom_components/systemd/hass.service`


If you're already using systemd to launch HA, you'll need to stop the existing
service first and replace it with the included file. This can be done as follows:

`sudo systemctl disable --now my-ha.service`

Now we'll copy and enable the new service:

`sudo cp /home/homeassistant/.homeassistant/custom_components/systemd/hass.service /etc/systemd/system/`

`sudo systemctl daemon-reload`

`sudo systemctl enable hass.service`


Now test the new service:

`sudo systemctl start hass.service; journalctl -f -u hass.service`

and monitor the journal and make sure no errors appear:
Wait for at least 5 minutes to make sure the watchdog is functioning.


`sudo systemctl start hass.service`

`sudo systemctl stop hass.service`

Verify the component is reporting status to systemd:

`sudo systemctl status hass.service`


Look for the 'Status:' line near the top, it should read "Home Assistant is running."


## Service File Options
### Required Options
The only options that *must* be set before using the service file are in the `[Service]` section:
- `Type`
  This option *must* be set to `notify`.
- `ExecStart`
  Command used to start HA.
- `User`
  Username to run HA under.

### After, Wants, Before
If you want to order other services to start before or after HA, you'll need to add their units to the
`Before=` or `After=` directives in the `[Unit]` section. Required dependencies should be added to the
`After=` *and* `Wants=` directives. Here are some tips:
- *Network*
  To delay HA until the network is up, add `network-online.target` to the `After=` and `Wants=` directives.
- *Time Sync*
  To make sure your system clock is set before HA starts, add `time-sync.target` to the `After=` and
  `Wants=` directives, then run `systemctl enable systemd-time-wait-sync.service` to enable the target.
- *Bluetooth*
  If you use BTLE for device tracking, add `bluetooth.target` to the `After=` and `Wants=` directives.
- *Databases*
  If you use an external database (Postgres, MySQL, etc.), make sure it starts first by adding
  the unit name to the `After=` and `Wants=` directives.
- *MQTT*
  If you run a local instance of Mosquitto, add `mosquitto.service` to the `After=` and `Wants=` directives.
### Timeouts
The various timeout directives in the `[Service]` section control how and when systemd restarts HA
on failures and errors.
- `TimeoutStartSec` & `TimeoutStopSec`
  Controls how long systemd waits for HA to complete startup and shutdown operations, respectivly.
- `WatchdogSec`
  The hass-systemd component must 'pet the dog' within this interval or systemd will kill HA. Comment
  this out to disable watchdog functionality.
- `Restart`
  Action to take when one of the above timeouts is reached. Default is to restart the service.
- `RestartSec`
  Delay between a timeout and performing the above action.


## Known Issues
Please report any issues here on GitHub. Enjoy!

## Version History
- 0.1.0 (2018.11.16)
  - Test release to verify concept.
- 0.1.1 (2018.11.17)
  - Switched to asyncio functions.
- 0.1.2 (2018.11.18)
  - Now reports the main PID to systemd.
  - Added comments and cleaned up the code.
- 0.2.0 (2019.04.26)
  - Converted to a HA integration.
  - Added manifest file.
  - Moved Function 'notify_status' inside 'async_setup'.

## Raspberry Pi setup
