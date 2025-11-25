# Setting up OctoPrint on a Raspberry Pi running Raspberry Pi OS (Debian)

**Important:** This guide expects you to have a more than basic grasp of the Linux command line. In order to follow it you'll need to know:

  * how to issue commands on the shell,
  * how to edit a text file from the command line,
  * what the difference is between your user account (e.g. `pi`) and the superuser account `root`,
  * how to SSH into your Pi (so you don't need to also attach keyboard and monitor),
  * how to use Git and
  * how to use the Internet to help you if you run into problems.

## Basic Installation

For the basic package you'll need Python 3.9, 3.10, 3.10, 3.11, 3.12 or 3.13 (one of these is probably installed by default) and pip.

Make sure you are using the correct version - it might be installed as `python3`, not `python`. To check:

```bash
python --version
```

and if that doesn't look fine

```bash
python3 --version
```

Installing OctoPrint should be done within a virtual environment, rather than an OS wide install, to help prevent dependency conflicts. To setup Python, dependencies and the virtual environment, run:

```bash
cd ~
sudo apt update
sudo apt install python3 python3-pip python3-dev python3-setuptools python3-venv git libyaml-dev build-essential libffi-dev libssl-dev
mkdir OctoPrint && cd OctoPrint
python3 -m venv venv
source venv/bin/activate
```

OctoPrint and it's Python dependencies can then be installed using `pip`:

```bash
pip install --upgrade pip wheel
pip install octoprint
```

> **Note:** If this installs an old version of OctoPrint, pip probably still has something cached. In that case add `--no-cache-dir` to the install command, e.g. `pip install --no-cache-dir octoprint`.

To make this permanent, clean `pip`'s cache:

```bash
rm -r ~/.cache/pip
```

You may need to add the pi user to the `dialout` group and `tty` so that the user can access the serial ports, before starting OctoPrint:

```bash
sudo usermod -a -G tty pi
sudo usermod -a -G dialout pi
```

> **Note:** You may have to log out and back in again for these changes to become effective.

## Starting the server for the first time

You should then be able to start the OctoPrint server using the `octoprint serve` command:

```bash
pi@raspberrypi:~ $ ~/OctoPrint/venv/bin/octoprint serve
2020-11-03 17:39:17,979 - octoprint.startup - INFO - ***************************
2020-11-03 17:39:17,980 - octoprint.startup - INFO - Starting OctoPrint 1.4.2
2020-11-03 17:39:17,980 - octoprint.startup - INFO - ***************************
```

Try it out\! Access the server by heading to `http://<pi's IP>:5000` and you should be greeted with the OctoPrint UI.

## Automatic start up

Create a file `/etc/systemd/system/octoprint.service` and put this into it:

```ini
[Unit]
Description=The snappy web interface for your 3D printer
After=network-online.target
Wants=network-online.target

[Service]
Environment="LC_ALL=C.UTF-8"
Environment="LANG=C.UTF-8"
Type=exec
User=pi
ExecStart=/home/pi/OctoPrint/venv/bin/octoprint serve

[Install]
WantedBy=multi-user.target
```

Adjust the paths to your `octoprint` binary as needed. If you set it up in a virtualenv as described above make sure your `/etc/systemd/system/octoprint.service` looks like this:

```ini
ExecStart=/home/pi/OctoPrint/venv/bin/octoprint
```

Then add the script to autostart using `sudo systemctl enable octoprint.service`.

This will also allow you to start/stop/restart the OctoPrint daemon via

```bash
sudo service octoprint {start|stop|restart}
```

## Make everything accessible on port 80

If you want to have nicer URLs or simply need OctoPrint to run on port 80 (http's default port) due to some network restrictions, I recommend using HAProxy as a reverse proxy instead of configuring OctoPrint to run on port 80.

Setup on Raspbian is as follows:

```bash
pi@raspberrypi ~ $ sudo apt install haproxy
```

I'm using the following configuration in `/etc/haproxy/haproxy.cfg`:

> **Warning:** Make sure you use the correct configuration for your version of Haproxy. Shown below is the configuration for Raspberry Pi OS Bullseye, which uses Haproxy 2.x and is the most up to date RPi OS.

### Haproxy 2.x (Debian 11, Bullseye etc.)

```haproxy
global
        maxconn 4096
        user haproxy
        group haproxy
        daemon
        log 127.0.0.1 local0 debug

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        option redispatch
        option http-server-close
        option forwardfor
        maxconn 2000
        timeout connect 5s
        timeout client  15min
        timeout server  15min

frontend public
        bind :::80 v4v6
        use_backend webcam if { path_beg /webcam/ }
        default_backend octoprint

backend octoprint
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        http-request replace-path /webcam/(.*)   /\1
        server webcam1 127.0.0.1:8080
```

### Haproxy 1.x (Debian 10, Buster, etc)

\<details\>
\<summary\>Click to expand Haproxy 1.x configuration\</summary\>

```haproxy
global
        maxconn 4096
        user haproxy
        group haproxy
        daemon
        log 127.0.0.1 local0 debug

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        option redispatch
        option http-server-close
        option forwardfor
        maxconn 2000
        timeout connect 5s
        timeout client  15min
        timeout server  15min

frontend public
        bind :::80 v4v6
        use_backend webcam if { path_beg /webcam/ }
        default_backend octoprint

backend octoprint
        reqrep ^([^\ :]*)\ /(.*)     \1\ /\2
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        reqrep ^([^\ :]*)\ /webcam/(.*)     \1\ /\2
        server webcam1 127.0.0.1:8080
```

\</details\>

This will make OctoPrint accessible under `http://<your Raspi's IP>/` and make mjpg-streamer accessible under `http://<your Raspi's IP>/webcam/`. You'll also need to modify `/etc/default/haproxy` and enable HAProxy by setting `ENABLED` to `1`. After that you can start HAProxy by issuing the following command:

```bash
sudo service haproxy start
```

Pointing your browser to `http://<your Raspi's IP>` should greet you with OctoPrint's UI. Now open the settings and switch to the webcam tab or alternatively open `~/.octoprint/config.yaml`. Set the webcam's stream URL from `http://<your Raspi's IP>:8080/?action=stream` to `/webcam/?action=stream` (leave the snapshotUrl at `http://127.0.0.1:8080/?action=snapshot`\!) and reload the page.

If everything works you can add the following lines to `~/.octoprint/config.yaml` (just create it if it doesn't exist yet) to make the server bind only to the loopback interface:

```yaml
server:
    host: 127.0.0.1
```

Restart the server. OctoPrint should still be available on port 80, including the webcam feed (if enabled).

## Updating & changing release channels & rolling back

OctoPrint should offer to update itself automatically. If you want or need to perform any of this manually, perform the following commands to install `<version>` of OctoPrint:

```bash
source ~/OctoPrint/venv/bin/activate
pip install octoprint==<version>
```

e.g.

```bash
source ~/OctoPrint/venv/bin/activate
pip install octoprint==1.4.0
```

## Support restart/shutdown through OctoPrint's system menu

In the UI, under **Settings \> Commands**, configure the following commands:

  * **Restart OctoPrint:** `sudo service octoprint restart`
  * **Restart system:** `sudo shutdown -r now`
  * **Shutdown system:** `sudo shutdown -h now`

> **Note:** If you disabled Raspbian's default behaviour of allowing the `pi` user passwordless sudo for every command, you'll need to explicitly allow the `pi` user passwordless sudo access to the `/sbin/shutdown` program.

Create a file `/etc/sudoers.d/octoprint-shutdown` (as root) with the following contents:

```text
pi ALL=NOPASSWD: /sbin/shutdown
```

Create another file `/etc/sudoers.d/octoprint-service` (as root) with the following contents:

```text
pi ALL=NOPASSWD: /usr/sbin/service
```

## Optional: Webcam

If you also want webcam and timelapse support, you'll need to download and compile MJPG-Streamer:

```bash
cd ~
sudo apt install subversion libjpeg62-turbo-dev imagemagick ffmpeg libv4l-dev cmake
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
export LD_LIBRARY_PATH=.
make
```

> **Heads-up:** The required packages depend on the underlying version of Debian\! The above is what should work on the current Debian Stretch or Buster based images of Raspbian.

For **Jessie** use:
`sudo apt install subversion libjpeg62-turbo-dev imagemagick libav-tools libv4l-dev cmake`

For **Wheezy or older** use:
`sudo apt install subversion libjpeg8-dev imagemagick libav-tools libv4l-dev cmake`

You should then be able to start the webcam server using:

```bash
./mjpg_streamer -i "./input_uvc.so" -o "./output_http.so"
```

For some webcams (including the PS3 Eye) you'll need to force the YUV mode:

```bash
./mjpg_streamer -i "./input_uvc.so -y" -o "./output_http.so"
```

If you want to use the official RaspberryPi Camera Module you need to run:

```bash
./mjpg_streamer -i "./input_raspicam.so -fps 5" -o "./output_http.so"
```

To configure the stream in OctoPrint, open the settings dialog and modify the following entries under **Webcam & Timelapse**:

  * **Stream URL:** `/webcam/?action=stream`
  * **Snapshot URL:** `http://127.0.0.1:8080/?action=snapshot`
  * **Path to FFMPEG:** `/usr/bin/ffmpeg`

> **Heads-up:** If you are using a Raspbian image based on Debian Jessie or older, "Path to FFMPEG" should instead be `/usr/bin/avconv`.

## Optional: Webcam Automatic Startup

If you want mjpg-streamer to automatically startup on boot:

Create a new file at `/home/pi/scripts/webcamDaemon` (ie. run `nano /home/pi/scripts/webcamDaemon`), with the following content:

```bash
#!/bin/bash

MJPGSTREAMER_HOME=/home/pi/mjpg-streamer/mjpg-streamer-experimental
MJPGSTREAMER_INPUT_USB="input_uvc.so"
MJPGSTREAMER_INPUT_RASPICAM="input_raspicam.so"

# init configuration
camera="auto"
camera_usb_options="-r 640x480 -f 10"
camera_raspi_options="-fps 10"

if [ -e "/boot/octopi.txt" ]; then
    source "/boot/octopi.txt"
fi

# runs MJPG Streamer, using the provided input plugin + configuration
function runMjpgStreamer {
    input=$1
    pushd $MJPGSTREAMER_HOME
    echo Running ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    LD_LIBRARY_PATH=. ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    popd
}

# starts up the RasPiCam
function startRaspi {
    logger "Starting Raspberry Pi camera"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_RASPICAM $camera_raspi_options"
}

# starts up the USB webcam
function startUsb {
    logger "Starting USB webcam"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_USB $camera_usb_options"
}

# we need this to prevent the later calls to vcgencmd from blocking
# I have no idea why, but that's how it is...
vcgencmd version

# echo configuration
echo camera: $camera
echo usb options: $camera_usb_options
echo raspi options: $camera_raspi_options

# keep mjpg streamer running if some camera is attached
while true; do
    if [ -e "/dev/video0" ] && { [ "$camera" = "auto" ] || [ "$camera" = "usb" ] ; }; then
        startUsb
    elif [ "`vcgencmd get_camera`" = "supported=1 detected=1" ] && { [ "$camera" = "auto" ] || [ "$camera" = "raspi" ] ; }; then
        startRaspi
    fi
    sleep 120
done
```

Make sure the file is executable:

```bash
chmod +x /home/pi/scripts/webcamDaemon
```

And then create another new file at `/etc/systemd/system/webcamd.service` (`sudo nano /etc/systemd/system/webcamd.service`), with these lines:

```ini
[Unit]
Description=Camera streamer for OctoPrint
After=network-online.target OctoPrint.service
Wants=network-online.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/scripts/webcamDaemon

[Install]
WantedBy=multi-user.target
```

Tell the system to read the new file:

```bash
sudo systemctl daemon-reload
```

And finally enable the service:

```bash
sudo systemctl enable webcamd
```

If you want to be able to start and stop the webcam server through OctoPrint's system menu, add the following to `.octoprint/config.yaml`:

```yaml
system:
  actions:
   - action: streamon
     command: sudo systemctl start webcamd
     confirm: false
     name: Start video stream
   - action: streamoff
     command: sudo systemctl stop webcamd
     confirm: false
     name: Stop video stream
```

> **Note:** mjpegstreamer does not allow to bind to a specific interface. If you want your octoprint instance to be reachable from the internet you need to block access to port 8080 from all sources except localhost using `iptables` rules.

## Optional: Touch UI

*Touch UI has been abandoned. Check here for potential updates.*

Install the plugin using the plugin manager. If you want to use this for a local LCD, first install `xautomation` and `epiphany-browser`:

```bash
sudo apt install epiphany-browser xautomation
```

Next, create a file `startTouchUI.sh` in `~/` and add:

```bash
#!/bin/bash
function check_octoprint {
    pgrep -n octoprint > /dev/null
    return $?
}

until check_octoprint
do
    sleep 5
done

sleep 5s
epiphany-browser http://127.0.0.1:5000 --display=:0 &
sleep 10s;
xte "key F11" -x:0
```

Make it executable: `chmod +x startTouchUI.sh` and add the following to `~/.config/lxsession/LXDE-pi/autostart`:

```text
@/home/pi/startTouchUI.sh
```

## Optional: Reach your printer by typing its name (Avahi)

Installation is simple, on your RasPi just type:

```bash
sudo apt update && sudo apt install avahi-daemon
```

The next step is to change the hostname of your RasPi into something more printer specific (e.g. `<yourprinter>`) via editing the files `/etc/hostname` and `/etc/hosts` on your RasPi.

Change the default name into `<yourprinter>` in the hostname-file via `sudo nano /etc/hostname` and do the same in the hosts-file via `sudo nano /etc/hosts`.

Now restart your RasPi via `sudo reboot`. You can now reach your RasPi running OctoPrint within your network by pointing your browser to `<yourprinter>.local`.