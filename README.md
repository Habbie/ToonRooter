# ToonRooter

## What it does
This application roots your Toon from a Raspberry Pi or another device with a JTAG debugger attached.

At the moment, the following is implemented:
 - Detection of the U-Boot version
 - Loading of modified U-Boot version using JTAG
 - Logging in to supported versions of U-Boot
 - Setting up the U-Boot environment so the Toon boots into a serial console
 - Install and set up dropbear and sftp-server to let root user log in using an ssh key
 - Modify the firewall settings to enable remote access to ssh and http server
 - Generate SSH keys or let the user supply their own

### What it might do in the more distant future
 - Check the output from a few commands so we know if we're successful...
 - More fine-grained control of the installation package

## How safe is it?

I don't know.

You'll probably be fine. We're not overwriting the bootloader or important parts
of the OS, so I don't see it really breaking anything, but you should make sure all wires are connected
correctly, because otherwise you might short out something and break your Pi, Toon or both. If you are
having issues with your Toon, run this application again with the `--boot-only` flag set. When it's finished
you should be able to access a serial console on the Toon and check out what's wrong. To find some
'usual suspects', look through `payload/patch_toon.sh`, which is the script that's actually run on the Toon
to enable SSH access. The rest of the application basically just opens an injection vector to make that
possible.

## I have a Toon 2 ##
The new Toon 2 isn't rootable using this tool. There are just a few people who can help you root a toon 2.
You can take a look at this forum post if you want to have your Toon 2 rooted: https://www.domoticaforum.eu/viewtopic.php?f=101&t=12162

## How to use it?

Make sure there's no power going in to either of the devices, and double check the connections
before powering up again.
Connect your Toon's debugging header to a Raspberry Pi according to the following pin assignments:

| Toon | Signal | Pi   |
|:----:|:------:|:----:|
|  1   |  RTCK  |      |
|  2   |  TRST  |  24  |
|  3   |  GND   |  25  |
|  4   |  TCK   |  23  |
|  5   |  GND   |  20  |
|  6   |  TMS   |  22  |
|  7   |  SRST  |  18  |
|  8   |  TDI   |  19  |
|  9   |  Vt    |      |
|  10  |  TDO   |  21  |
|  11  |  RxD   |  8   |
|  12  |        |      |
|  13  |  TxD   |  10  |
|  14  |  GND   |  9   |

If you don't know which pins on the Pi are which, see [pinout.xyz](https://pinout.xyz). The pins on the Toon are numbered similarly to the Pi, starting at the one marked as 1.

Check if you're running Raspbian Stretch or later. You can check this by running 
`cat /etc/issue`, the response should be `Raspbian GNU/Linux 9 \n \l` or later.
Then make sure the serial port on the Pi is enabled and the serial console is disabled
using `sudo raspi-config` and reboot if necessary. Install the dependencies mentioned in the
[Dependencies](#dependencies)-section.

Then get and run this application:
```bash
sudo apt install python-serial python-cryptography
git clone https://github.com/martenjacobs/ToonRooter.git
cd ToonRooter
sudo python . --jtag-available
```

Then reset your Toon and let the magic happen :)

## Logging in to the Toon after rooting

During the rooting process, the script will generate a random root password and output it to 
the console. It will also generate an ssh private key. You can use either to log in to the 
Toon after it is rooted. The script does not use a generic default password for all Toons 
because it's considered a security hazard (and there's no easy way to change the password on 
the Toon itself). 

The most secure way of logging in is using the SSH private key (even better if you encrypted 
the key using the `--private-key-password` argument). 

After rooting, a file called `id_rsa` should be present in your working directory (or something 
else if you used `--output-ssh-key`). 
Logging in to the Toon can then be done by running:
```bash 
ssh -i id_rsa root@[TOON IP]
```
If you used the `--private-key-password` argument, you will now be asked to insert the password
you provided there. If you didn't, it should log in without asking for a password at all. 

In the latter case, please consider encrypting it later using the following command:
```bash
ssh-keygen -p -f id_rsa
```

If for some reason you lost both the ssh private key and the password, you'll have to 
re-root your Toon.

## It's not working!
Please re-check your wiring. If you're sure the wiring is correct, try the command with 
the `--output-level DEBUG` flag set and head over to 
[this friendly forum](https://www.domoticaforum.eu/viewtopic.php?f=101&t=11999) where the
issue has most likely already been solved. If not, post a reply and the active community
will probably help you out.

## But I don't have a Pi

You should definitely get a Pi.

However, if you're adamant that you want to root your Toon from another device and
you have a JTAG debugger lying around that works with OpenOCD, you should be able to
use this script without issue. Just put the configuration file for your debugger in the
`assets/adapters` directory (make sure it has a `.cfg` extension) and pass the name
of the file (without extension) to the script using the `--jtag-hardware` argument.
I'm pretty sure Windows is not going to work though, so you should use a Linux
(virtual) machine.

## Command line arguments

```
usage: sudo python . [-h] [--serial-port PATH] [--ssh-public-key PATH]
                  [--output-ssh-key PATH] [--private-key-password PASSWORD]
                  [--output-level INFO|DEBUG] [--jtag-available]
                  [--dont-check-uboot] [--dont-cleanup-payload]
                  [--dont-reboot-after] [--boot-only]

Root your Toon.

optional arguments:
  -h, --help            show this help message and exit
  --serial-port PATH    The path of the serial port to use
  --ssh-public-key PATH
                        The path to an RSA public key which should be an
                        allowed key on the Toon after rooting it. If not
                        supplied, a key pair is generated
  --output-ssh-key PATH
                        The path to output a generated key pair (the public
                        key will have a .pub suffix). Default is to write
                        id_rsa and id_rsa.pub in the current directory.
                        Ignored if you've used --ssh-public-key
  --private-key-password PASSWORD
                        The password that should be used to encrypt the
                        private key when it's generated. Default is to use no
                        encryption. Ignored if you've used --ssh-public-key
  --output-level INFO|DEBUG
                        The level of output to print to the console
  --jtag-available      Indicates you have a JTAG debugger connected to your
                        Toon's JTAG headers
  --jtag-hardware TYPE  The JTAG debugger type that we're working with. The
                        default is to autodetect the JTAG debugger (which
                        currently only works on Raspberry Pi). Supported
                        values are: auto, rpi1, rpi2, rpi3
  --dont-check-uboot    Don't check whether we can access the installer
                        version of U-Boot before using JTAG to start up the
                        custom one.
  --dont-cleanup-payload
                        Leave the payload in /payload. Use this if you want to
                        include more files and do something with them.
  --dont-reboot-after   Don't reboot the Toon after rooting it. Use this if
                        you want to use the serial console after rooting
  --uboot-only          Only boot to the u-boot environment for manual control
  --boot-only           Don't install the payload, just boot into the serial
                        console
```

## Dependencies

- Raspbian Stretch (or later) if you're using a Pi
- Python 2.7
- pySerial 3.4 (or later)
- OpenOCD from git (for newer Toons) (see [instructions](#install-openocd))

## Install OpenOCD
If your Toon has a newer U-Boot version than 2010-R8, a JTAG interface is required to
upload a bootloader that we have access to through the serial console. To do this,
you need to build a version of OpenOCD (at the time of writing the version in apt
doesn't support using the Pi's headers as JTAG debugger).

```bash
git clone --recursive git://git.code.sf.net/p/openocd/code openocd
cd openocd
sudo apt install make libtool libtool-bin pkg-config autoconf automake texinfo libusb-1.0 libusb-dev
./bootstrap
./configure --enable-sysfsgpio --enable-bcm2835gpio --prefix=/usr
make -j4
sudo make install
```
> these instructions were based on the instructions posted [here](https://www.domoticaforum.eu/viewtopic.php?f=87&t=11230&start=210#p83745) by rboers


## It's not working!
Common mistakes while using this tool is first of all just please read all information above carefully :-)

Always re-check your wiring. A lot of people just forget to wire properly.

Also a common mistake is that the /dev/serial0 port isn't enabled (read above) or the Pi console is still connected to the serial port (resulting in double read/writes to the serial port), also read above how to fix this.

Some people have unknown raspberry pi versions, mainly because of of the overclock/warranty bit has been triggered. This tool can't identify your Pi version because of this so you need to specify it manually.

If you still can't figure out what is going wrong try the command with 
the `--output-level DEBUG` flag set and head over to
[this friendly forum](https://www.domoticaforum.eu/viewtopic.php?f=101&t=11999) where the
issue has most likely already been solved. If not, post a reply and the active community
will probably help you out.

## It's is rooted, but what now?
If the tool worked and you got your SSH access (using the generated root password or ssh key) and are able to run commands on the Toon linux console then you probably want to have those pretty rooted Toon modifications and want to close down your Toon from Eneco and Quby. Just head to the update-rooted script on https://github.com/ToonSoftwareCollective/update-rooted and run that script on your freshly rooted Toon. With option -f it will only run the necessary stuff to setup your Toon properly as a mainstream rooted Toon. Or just run the script without options which will guide you into updating the Toon to the latest firmware first and finish with installing the necessary stuff.


## Thanks
This application is based on instructions and software written by:
- marcelr
- thehognl
- klaphekje
- rboers
- other [domoticaforum.eu](https://www.domoticaforum.eu/viewtopic.php?f=101) users
