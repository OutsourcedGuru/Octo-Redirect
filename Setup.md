# Octo-Redirect
Step-by-step instructions for setting up a Raspberry Pi 3B and a Raspberry Pi Zero W to remotely print.

## Overview
The following diagram roughly describes the connections and communications required.

| Raspi3B | ➡️ | ZeroW | with | rfc2217_server.py | ➡️ | Mega2560 |
|---|---|---|---|---|---|---|
| rfc2217://10.20.30.66:2217 | tcp | 10.20.30.66:2217 | rfc2217 port forwarding to serial | `/dev/ttyACM0` on microUSB with Type A OTG adapter cable | USB serial cable | Type B connector |

Feel free to adjust this to use your own private IP address space, of course.

## Minimum parts list
* Raspberry Pi 3B with OctoPi image installed
* Raspberry Pi Zero W with Raspbian Stretch Lite installed
* Two 8GB microSD adapters
* Two 5V 2.5A power adapters
* One microUSB male to Type A female adapter (I'm using an OTG adapter cable)
* One Type A to Type B USB shielded serial cable
* One printer controller board with Marlin or similar firmware
* One power adapter for the printer controller board

## Setting up the Zero
The following script comes from the [Pyserial](https://github.com/pyserial/pyserial/blob/master/examples/rfc2217_server.py) repository's examples.

For ESP8266 see also [esp-link](https://github.com/jeelabs/esp-link).

```
sudo apt-get install -y python-pip
pip install pyserial
mkdir ~/SerialRedirect && cd ~/SerialRedirect
touch ./rfc2217_server.py && chmod a+x ./rfc2217_server.py
nano ./rfc2217_server.py
```

### rfc2217_server.py:
```
#!/usr/bin/env python
#
# redirect data from a TCP/IP connection to a serial port and vice versa
# using RFC 2217
#
# (C) 2009-2015 Chris Liechti <cliechti@gmx.net>
#
# SPDX-License-Identifier:    BSD-3-Clause

import logging
import socket
import sys
import time
import threading
import serial
import serial.rfc2217


class Redirector(object):
    def __init__(self, serial_instance, socket, debug=False):
        self.serial = serial_instance
        self.socket = socket
        self._write_lock = threading.Lock()
        self.rfc2217 = serial.rfc2217.PortManager(
            self.serial,
            self,
            logger=logging.getLogger('rfc2217.server') if debug else None)
        self.log = logging.getLogger('redirector')

    def statusline_poller(self):
        self.log.debug('status line poll thread started')
        while self.alive:
            time.sleep(1)
            self.rfc2217.check_modem_lines()
        self.log.debug('status line poll thread terminated')

    def shortcircuit(self):
        """connect the serial port to the TCP port by copying everything
           from one side to the other"""
        self.alive = True
        self.thread_read = threading.Thread(target=self.reader)
        self.thread_read.daemon = True
        self.thread_read.name = 'serial->socket'
        self.thread_read.start()
        self.thread_poll = threading.Thread(target=self.statusline_poller)
        self.thread_poll.daemon = True
        self.thread_poll.name = 'status line poll'
        self.thread_poll.start()
        self.writer()

    def reader(self):
        """loop forever and copy serial->socket"""
        self.log.debug('reader thread started')
        while self.alive:
            try:
                data = self.serial.read(self.serial.in_waiting or 1)
                if data:
                    # escape outgoing data when needed (Telnet IAC (0xff) character)
                    self.write(b''.join(self.rfc2217.escape(data)))
            except socket.error as msg:
                self.log.error('{}'.format(msg))
                # probably got disconnected
                break
        self.alive = False
        self.log.debug('reader thread terminated')

    def write(self, data):
        """thread safe socket write with no data escaping. used to send telnet stuff"""
        with self._write_lock:
            self.socket.sendall(data)

    def writer(self):
        """loop forever and copy socket->serial"""
        while self.alive:
            try:
                data = self.socket.recv(1024)
                if not data:
                    break
                self.serial.write(b''.join(self.rfc2217.filter(data)))
            except socket.error as msg:
                self.log.error('{}'.format(msg))
                # probably got disconnected
                break
        self.stop()

    def stop(self):
        """Stop copying"""
        self.log.debug('stopping')
        if self.alive:
            self.alive = False
            self.thread_read.join()
            self.thread_poll.join()


if __name__ == '__main__':
    import argparse

    parser = argparse.ArgumentParser(
        description="RFC 2217 Serial to Network (TCP/IP) redirector.",
        epilog="""\
NOTE: no security measures are implemented. Anyone can remotely connect
to this service over the network.

Only one connection at once is supported. When the connection is terminated
it waits for the next connect.
""")

    parser.add_argument('SERIALPORT')

    parser.add_argument(
        '-p', '--localport',
        type=int,
        help='local TCP port, default: %(default)s',
        metavar='TCPPORT',
        default=2217)

    parser.add_argument(
        '-v', '--verbose',
        dest='verbosity',
        action='count',
        help='print more diagnostic messages (option can be given multiple times)',
        default=0)

    args = parser.parse_args()

    if args.verbosity > 3:
        args.verbosity = 3
    level = (logging.WARNING,
             logging.INFO,
             logging.DEBUG,
             logging.NOTSET)[args.verbosity]
    logging.basicConfig(level=logging.INFO)
    #~ logging.getLogger('root').setLevel(logging.INFO)
    logging.getLogger('rfc2217').setLevel(level)

    # connect to serial port
    ser = serial.serial_for_url(args.SERIALPORT, do_not_open=True)
    ser.timeout = 3     # required so that the reader thread can exit
    # reset control line as no _remote_ "terminal" has been connected yet
    ser.dtr = False
    ser.rts = False

    logging.info("RFC 2217 TCP/IP to Serial redirector - type Ctrl-C / BREAK to quit")

    try:
        ser.open()
    except serial.SerialException as e:
        logging.error("Could not open serial port {}: {}".format(ser.name, e))
        sys.exit(1)

    logging.info("Serving serial port: {}".format(ser.name))
    settings = ser.get_settings()

    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(('', args.localport))
    srv.listen(1)
    logging.info("TCP/IP port: {}".format(args.localport))
    while True:
        try:
            client_socket, addr = srv.accept()
            logging.info('Connected by {}:{}'.format(addr[0], addr[1]))
            client_socket.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
            ser.rts = True
            ser.dtr = True
            # enter network <-> serial loop
            r = Redirector(
                ser,
                client_socket,
                args.verbosity > 0)
            try:
                r.shortcircuit()
            finally:
                logging.info('Disconnected')
                r.stop()
                client_socket.close()
                ser.dtr = False
                ser.rts = False
                # Restore port settings (may have been changed by RFC 2217
                # capable client)
                ser.apply_settings(settings)
        except KeyboardInterrupt:
            sys.stdout.write('\n')
            break
        except socket.error as msg:
            logging.error(str(msg))

    logging.info('--- exit ---')
```

### Plug in the MEGA board
```
dmesg

[  799.956437] Indeed it is in host mode hprt0 = 00021501
[  800.166345] usb 1-1: new full-speed USB device number 2 using dwc_otg
[  800.166702] Indeed it is in host mode hprt0 = 00021501
[  800.408065] usb 1-1: New USB device found, idVendor=2341, idProduct=0010
[  800.408082] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=220
[  800.408089] usb 1-1: Product: Arduino Mega 2560
[  800.408096] usb 1-1: Manufacturer: Arduino (www.arduino.cc)
[  800.408103] usb 1-1: SerialNumber: 75633313233351610211
[  800.522349] cdc_acm 1-1:1.0: ttyACM0: USB ACM device
[  800.524405] usbcore: registered new interface driver cdc_acm
[  800.524417] cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters
```

I note that the new device is `/dev/ttyACM0` in this case.

`sudo raspi-config`
* Auto-boot to the console as `pi`
* Localisation: TZ, etc
* Hostname: octo-redirect1
* GPU memory: 16

Reboot and reconnect with the new hostname.

### Now run the service
```
cd ~/SerialRedirect
./rfc2217_server.py /dev/ttyACM0

INFO:root:RFC 2217 TCP/IP to Serial redirector - type Ctrl-C / BREAK to quit
INFO:root:Serving serial port: /dev/ttyACM0
INFO:root:TCP/IP port: 2217
```

## Raspi3B
The following comes from [this conversation](https://github.com/foosel/OctoPrint/issues/3070) on OctoPrint's Issues collection.

### Prep
Create a vanilla OctoPi image from 0.16.0 on a Raspberry Pi 3B.

#### Change this to use a check-out version of OctoPrint, noting that you can't be in the `~/OctoPrint` folder for this to be happy.
```
~/scripts/add-octoprint-checkout

...
Your git checkout is now available at ~/OctoPrint. Please note that it
is currently not installed. If you want to replace the default installation
of OctoPrint with whatever is currently checked out in your git checkout
you'll need to do this manually. You'll also need to keep your checkout
up to date manually if you still have OctoPrint's update mode set to release
tracking.

sudo service octoprint stop
cd ~/OctoPrint
virtualenv venv
source venv/bin/activate
```

### Edit the default OctoPrint code

It suggests a change to [OctoPrint/src/octoprint/util/comm.py](https://github.com/foosel/OctoPrint/blob/master/src/octoprint/util/comm.py)

Line 2442 `def _openSerial(self):` section:
Line 2454:

```
sudo nano ~/OctoPrint/src/octoprint/util/comm.py
```

### Before

```
			# connect to regular serial port
			self._log("Connecting to: %s" % port)
			if baudrate == 0:
				baudrates = baudrateList()
				serial_obj = serial.Serial(str(port),
				                           baudrates[0],
				                           timeout=read_timeout,
				                           write_timeout=10000,
				                           parity=serial.PARITY_ODD)
			else:
				serial_obj = serial.Serial(str(port),
				                           baudrate,
				                           timeout=read_timeout,
				                           write_timeout=10000,
				                           parity=serial.PARITY_ODD)
```

### After

```
			# connect to regular serial port
			self._log("Connecting to: %s" % port)
			if '://' in str(port):                            # Begin -----
				serial_obj = serial.serial_for_url(str(port)) # -----------
			else:                                             # End -------
				if baudrate == 0:                             # Indent this
					baudrates = baudrateList()
					serial_obj = serial.Serial(str(port),
					                           baudrates[0],
					                           timeout=read_timeout,
					                           write_timeout=10000,
					                           parity=serial.PARITY_ODD)
				else:
					serial_obj = serial.Serial(str(port),
					                           baudrate,
					                           timeout=read_timeout,
					                           write_timeout=10000,
					                           parity=serial.PARITY_ODD)			
```

Line 156 `def serialList():` section:

### Before

```
	additionalPorts = settings().get(["serial", "additionalPorts"])
	for additional in additionalPorts:
		baselist += glob.glob(additional)
```

### After

```
	additionalPorts = settings().get(["serial", "additionalPorts"])
	for additional in additionalPorts:
		if '://' in additional:                              # Begin -----
			baselist.append(additional)                      # -----------
		else:                                                # End -------
			baselist += glob.glob(additional)                # Indent this
```

#### Having edited the OctoPrint code in-place, then install it:
```
pip install -e .[develop,plugins]
pip freeze|grep OctoPrint

# -e git+https://github.com/foosel/OctoPrint.git@8ce2290d63ed395ef1b1caa74537a517eac9efb2#egg=OctoPrint
# We want to see something like this instead of just "octoprint" or similar which
# would be the standard installation.
```

Note that if you update your OctoPrint's version, it will overwrite these patching attempts.

#### Run it

```
octoprint serve
```

### Settings update
Visit the OctoPrint web interface and make the following edit:

* OctoPrint -> Settings -> Serial -> Additional Serial Port = `rfc2217://10.20.30.66:2217`
* OctoPrint -> Connection side panel -> Connect
* OctoPrint -> Terminal tab (observe)
* OctoPrint -> upload a dry-run gcode file (10mm cube) and print it

Let this run all the way through.

## Performance
I include some comparison information between using a Zero W and a 3A+ computer as the receiver. I used a standard 10mm cube file which has been modified for "dry-run" (no heat/extrusions/fans).

| 10mmCube | ZeroW | 3A+ | Notes |
|---|---|---|---|
| Cores | 1 | 4 | |
| ProcSpeed | 1GHz | 1.4GHz | |
| RAM | 512MB | 512MB | |
| gpu_mem set | 16MB | 16MB | |
| Wi-fi | 2.4GHz | 5GHz&nbsp;&&nbsp;2.4GHz | Connected over 5GHz for 3A+|
| USB | microUSB | Type A | ZeroW required adapter for serial cable|
| **Lines&nbsp;of&nbsp;gcode** | **82010** | **82010** | Filesize 1.6MB |
| **Time&nbsp;to&nbsp;print** | **55:32** | **50:44** | **Appears to be 9% faster on 3A+** |
| **Avg&nbsp;lines/sec** | **24.613** | **26.941** | "82010 / ((55 * 60) + 32)", e.g. |
| **Avg&nbsp;line&nbsp;length** | **19.72** | **19.72** | "1617160 / 82010" |
| **Avg&nbsp;chars/sec** | **485.368** | **531.277** | "24.613 * 19.72", e.g. |
| **Avg&nbsp;bits/sec** | **3882.944** | **4250.216** | "485.368 * 8", e.g. |
| Controller board | MEGA2560 | MEGA2560 | Robo 3D OEM version |
| Baudrate | 115200 | 115200 | It would be interesting to attempt to flash a new version of the Marlin firmware to increase the baudrate|
