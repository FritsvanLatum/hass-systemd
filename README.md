# hass-systemd v0.2.1


Install Home Assistant Core on linux, e.g. on a Raspberry Pi, and run Home Assistant as a systemd notify daemon,
with watchdog support.

(C) Timothy Brown 2018.11.18    
(C) Frits  van Latum 2023.02.07

## Instructions

### Raspberry Pi

These instructions assume you're working on a Raspberry Pi, with **Raspberry PI OS** already installed. 
See [Raspberry Pi setup](#Raspberry%20Pi%20setup) for some tips about setting up your Raspberry Pi.

### Home Assistant Installation

The installation procedure of **Home Assistant Core** as decribed in
[Install Home Assistant Core](https://www.home-assistant.io/installation/linux#install-home-assistant-core) is **NOT** 
complete. Two dependencies are missing. You can add these after installing the list of dependenncies and just before
creating the account `homeassistant`:

`sudo apt-get install -y rustc`

The rustc compiler is used to install Home Assistant's cryptography module.

`sudo apt-get install -y libatlas-base-dev`

This is an addon needed by the numpy module.


Furthermore, at the end of the procedure, the installation command itself:

`pip3 install homeassistant==2023.2.x`

frequently results in an error message saying that the (latest) version cannot (yet) be found in the pip repository.
This is the work around:

`pip3 install homeassistant`

which installs the latest available version. The installation can last more than 10 minutes, so be (very) patient.

After the installation:

- you have created a user `homeassistant`
- this user has `\home\homeassistant` as its home directory
- you have created a the directory `\srv\homeassistant`
- and installed the Home Assistant software in a Python3 virtual environment
- while autmatically a configuration directory `/home/homeassistant/.homeassistant` has been created.

According to the installation instructions Home Assistant is now started by:

`hass`

keeping the terminal window open...

When you want to start Home Assistant again, for instance after closing your terminal window, you have to 
use the following commands:

```
sudo -u homeassistant -H -s
cd /srv/homeassistant
source bin/activate
hass
```

meaning: start a new session logged in as `homeassistant`, goto the installation directory, activate the
virtual environment and start Home Assistant with `hass`.

### Running Home Assistant automatically, as a deamon

Now we want to run Home Assistant automatically, without a terminal. It should also start automatically when
the Raspberry Pi is started. More in detail: we want Home Assistant to run as a systemd notify daemon, with
watchdog support.

The linux operating system of your Raspberry Pi must have `systemd`. 

These are the instructions:
 

Create the custom components directory, if needed:

`sudo -u homeassistant mkdir -p /home/homeassistant/.homeassistant/custom_components`


Download the plugin files:

```
sudo git clone https://github.com/FritsvanLatum/hass-systemd.git     
        /home/homeassistant/.homeassistant/custom_components/systemd
```


Set file permissions:

`sudo chown -R homeassistant:homeassistant /home/homeassistant/.homeassistant/custom_components/systemd`


Edit your HA configuration to enable our new component:

`sudo -u homeassistant nano /home/homeassistant/.homeassistant/configuration.yaml`


and add the following line somewhere in the file:
    
`systemd:`


If your Home Assistant user is not called `homeassistant` or you have chosen another installation directory for
Home Assistant, you have to edit the systemd service file:

`sudo -u homeassistant nano /home/homeassistant/.homeassistant/custom_components/systemd/hass.service`

and make changes to the `ExecStart=` and `User=` lines.


Now we are going to setup the systemd service. First make `root` the owner of the service file:

`sudo chown root:root /home/homeassistant/.homeassistant/custom_components/systemd/hass.service`


If you're already using systemd to launch HA, you'll need to stop the existing
service first and replace it with the included file. This can be done as follows:

`sudo systemctl disable --now my-ha.service`

Now we'll copy and enable the new service:

```
sudo cp /home/homeassistant/.homeassistant/custom_components/systemd/hass.service /etc/systemd/system/
sudo systemctl daemon-reload`
sudo systemctl enable hass.service
```


Now test the new service:

`sudo systemctl start hass.service; journalctl -f -u hass.service`

monitor the journal and make sure no errors appear. 
Wait for at least 5 minutes to make sure the watchdog is functioning.

Interupt the journal. You can stop, start and restart the service:

```
sudo systemctl stop hass.service
sudo systemctl start hass.service
sudo systemctl restart hass.service
```

You can also check the status of the service:

`sudo systemctl status hass.service`

Look for the 'Status:' line near the top, it should read "Home Assistant is running."


## Service File Options
### Required Options

The only options that *must* be set before using the service file are in the `[Service]` section:

```
Type=notify
User=homeassistant
ExecStart=/srv/homeassistant/bin/hass -c "/home/homeassistant/.homeassistant"
```

### After, Wants, Before

If you want to order other services to start before or after HA, you'll need to add their units to the
`Before=` or `After=` directives in the `[Unit]` section. Required dependencies should be added to the
`After=` *and* `Wants=` directives. Here are some tips:

*Network*     
To delay HA until the network is up, add `network-online.target` to the `After=` and `Wants=` directives.
This is already done in `hass.service` in this repository.

*Time Sync*     
To make sure your system clock is set before HA starts, add `time-sync.target` to the `After=` and
`Wants=` directives, then run `systemctl enable systemd-time-wait-sync.service` to enable the target.

*Bluetooth*     
If you use BTLE for device tracking, add `bluetooth.target` to the `After=` and `Wants=` directives.

*Databases*     
If you use an external database (Postgres, MySQL, etc.), make sure it starts first by adding
the unit name to the `After=` and `Wants=` directives.

*MQTT*     
If you run a local instance of Mosquitto, add `mosquitto.service` to the `After=` and `Wants=` directives.


### Timeouts
The various timeout directives in the `[Service]` section control how and when systemd restarts HA
on failures and errors.

These directives are included:

```
TimeoutStartSec=5m
TimeoutStopSec=5m
WatchdogSec=5m
Restart=on-failure
RestartSec=15s
```

`TimeoutStartSec` & `TimeoutStopSec`     
Controls how long systemd waits for HA to complete startup and shutdown operations, respectivly.

`WatchdogSec`     
The hass-systemd component must 'pet the dog' within this interval or systemd will kill HA. Comment
this out to disable watchdog functionality.

`Restart`     
Action to take when one of the above timeouts is reached. Default is to restart the service.

`RestartSec`     
Delay between a timeout and performing the above action.


## Known Issues
Please report any issues here on GitHub. Enjoy!

  
## Raspberry Pi setup




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
- 0.2.1 (2023-02-09)
  - Edited the readme file (this text)
  - Edited the service file accordingly
  - Added Version to the manifest json file
