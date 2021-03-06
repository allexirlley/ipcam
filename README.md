# IP Cam using Telegram as DDNS

**Based on: Raspberry Pi 3 + camera module, Raspbian Stretch, Python 3.5**  
Setting: Pi → router → modem (which should give the router a publicly-accessible IP address)

You will learn:
- to stream MJPEG video from Raspberry Pi
- to open a port through the router to Raspberry Pi
- to setup a Telegram bot on Raspberry Pi and use it to communicate the router's
  IP address to you, so you can view the video stream away from home

**This system is intended for personal use and is not secure. Its purpose is
purely educational.**

## Enable Camera

```
sudo apt-get update
sudo apt-get upgrade
```

`sudo raspi-config`, select **Interfacing Options** and enable **Camera**. Then,
finish and **reboot**.

## Install mjpg_streamer

```
sudo apt-get install subversion libjpeg8-dev imagemagick libav-tools cmake
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
make
```

Run:
```
./mjpg_streamer -i "./input_raspicam.so -fps 5" -o "./output_http.so"
```

You should be able to view the stream by pointing your browser or VLC player to:  
`http://<Pi's IP>:8080/?action=stream`

[A lot of
options](https://github.com/foosel/OctoPrint/wiki/MJPG-Streamer-configuration)
[can be set from the
command-line](http://skillfulness.blogspot.hk/2010/03/mjpg-streamer-documentation.html).
For example,  to make it 10 frames-per-second (fps) and flip the horizontal and
vertical direction, you can do:

```
./mjpg_streamer -i "./input_raspicam.so -fps 10 -hf -vf" -o "./output_http.so"
```

-----
As a side note, you can tell mjpg_streamer to take a snapshot. For example, try
this in the browser:  
`http://<Pi's IP>:8080/?action=snapshot`

Or try this on another Raspberry Pi:  
`wget http://<Pi's IP>:8080/?action=snapshot -O output.jpg`

-----
I have also considered **VLC** and **Motion** as the streaming software, but
they generally stream VERY slowly (i.e. VERY long lag time) over 4G networks.
They don't provide an easy way to control parameters, like frame rate and
picture quality, that affect bandwidth usage.

Formats other than MJPEG may stream faster, but cannot be easily viewed on a
browser and may not be viewable on all mobile devices. MJPEG may be primitive,
but it is the most universal.

All factors considered, **mjpg_streamer** is the best option.

## mjpg_streamer as systemd service

To make it easy to start/stop mjpg_streamer, it is useful to create a systemd
service:

```
cd /lib/systemd/system
sudo nano mjpg_streamer.service
```

Insert the following contents. Note that I have added a few flags (`-quality 10
-x 400 -y 300`) to lessen bandwidth usage. These are realistic settings for
streaming over the internet.

```
[Unit]
Description=MJPG Streamer
After=network.target

[Service]
WorkingDirectory=/home/pi/mjpg-streamer/mjpg-streamer-experimental
ExecStart=/home/pi/mjpg-streamer/mjpg-streamer-experimental/mjpg_streamer \
                  -i "./input_raspicam.so -fps 5 -quality 10 -x 400 -y 300" \
                  -o "./output_http.so"

[Install]
WantedBy=multi-user.target
```

`sudo systemctl start mjpg_streamer.service` to start it.

`sudo systemctl stop mjpg_streamer.service` to stop it.

`sudo systemctl status mjpg_streamer.service` to view its status.

## Install miniupnpc

```
sudo apt-get install miniupnpc
```

After this, you should have the `upnpc` command installed. `upnpc` stands for
Universal Plug-n-Play Client, able to list, create, and delete port-forwards in
the router. Port-forward basically links a router's port with a Raspberry Pi's
port. For example, if you forward the router's port `54321` to Pi's port `8080`,
when someone visits the router port `54321`, he will be led to Pi's port `8080`.
If `mjpg_streamer` is also running, the video stream is exposed (which is what
you want from an IP cam).

I would not detail the `upnpc` command here. It is part of the [MiniUPnP
project](http://miniupnp.free.fr/).
[Read](http://www.makelinux.com/man/1/U/upnpc)
[more](http://superuser.com/questions/192132/how-to-automatically-forward-a-port-from-the-router-to-a-mac-upnp)
[about it](https://forum.transmissionbt.com/viewtopic.php?t=15840) [if you
want](http://po-ru.com/diary/using-upnp-igd-for-simpler-port-forwarding/).

To list existing port-forwards:

```
upnpc -l
```

**Pay attention to `ExternalIPAddress` in the output. It is your router's IP
address facing outside. You will have to be able to access this address if you
want to see the video stream away from home.**

To forward router's port `54321` → Pi's port `8080`:

```
upnpc -a <Pi's INTERNAL IP> 8080 54321 TCP
```

Assuming *mjpg_streamer is running* and router's *external IP address is
accessible*, you should be able to view the video stream in a browser:

```
http://<Router EXTERNAL IP>:54321/?action=stream
```

If the router's IP address stays constant, we could stop right now and the IP
Cam project finished. However, the IP address is assigned by the ISP and does
not stay the same. We need a way to communicate the most current IP address.

The usual way to do this is to get a domain name, and use a Dynamic Domain Name
Service (DDNS) to keep the IP address up-to-date. Here, I choose another
approach - **use Telegram to "simulate" a DDNS**. **I make a Telegram Bot
capable of reporting its IP address whenever you need it.**

Before moving on, you may want to delete the port-forward to hide the video
stream from the outside world:

```
upnpc -d 54321 TCP
```

## Get a Telegram Bot account

I have written a lot about [Telegram Bot API](https://core.telegram.org/bots):

- [How to setup a Telegram Bot on Raspberry Pi](http://www.instructables.com/id/Set-up-Telegram-Bot-on-Raspberry-Pi/)
- **[telepot](https://github.com/nickoala/telepot)**: a Python framework for Telegram Bot API

Obtain a bot token by [chatting with BotFather](https://core.telegram.org/bots).
And install the telepot package:

```
sudo pip3 install telepot
```

## Download this project

The goal is to run the
[ipcam.py](https://github.com/nickoala/ipcam/blob/master/ipcam.py) script. It
relies on some shell scripts provided with this project.

```
cd ~
wget https://github.com/nickoala/ipcam/archive/master.zip
unzip master.zip
mv ipcam-master ipcam
```

Inside `ipcam/scripts`, there are some shell scripts. They mostly enhance the
usability of `upnpc` and provide more readable output:

- `pf`: create and delete port-forwards to local Raspberry Pi
- `lspf`: List all port-forwards to local Raspberry Pi
- `ipaddr`: Extract all relevant IP addresses

Use `chmod` to make them executable, then move them to `/usr/local/bin`, so
they can be executed like normal Linux commands:

```
cd ipcam/scripts
chmod +x *
sudo cp * /usr/local/bin
```

## Run the bot

```
python3 ~/ipcam/ipcam.py <token>
```

- On startup, it starts mjpg_streamer. No router port is open yet, so the video
  stream is not accessible from outside.
- On receiving `/open`, it opens a port (default: 54321) through the router and
  text you the URL.
- On receiving `/close`, it closes the port on the router, so the video stream
  is no longer accessible from outside.

![](https://github.com/nickoala/ipcam/blob/master/images/ipcam.png?raw=true)

## Bonus

As a bonus, I also have a script,
[whatsmyip.py](https://github.com/nickoala/ipcam/blob/master/whatsmyip.py), a
pure IP-reporting Telegram Bot. Send it any messages, it answers with its public
IP address. If you have any servers running at home, this can be a handy tool.
