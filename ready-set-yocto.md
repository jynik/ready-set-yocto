# Ready, Set, Yocto! #

***A short, unofficial guide on getting started with Yocto Release 2.5 (Sumo) using a Raspberry Pi Zero (W)***

**Revision**: 2.5

**Author**: Jon Szymaniak ("jynik")

**License**: [CC BY-SA 2.0](https://creativecommons.org/licenses/by-sa/2.0/)

# Introduction #

If you're looking to build Linux-based firmware for embedded platforms, Yocto
is a good option for you. 

If you're looking to create reproducible and version-controlled automated 
firmware builds with excellent traceability, change tracking, license auditing,
and configurable QA tests... Yocto is a great option for you.

The purpose of this guide is to get you up and running with a minimal build for
a Raspberry Pi Zero so that you can dive into the documentation while you wait. 

Keep the following at an arm's length:

* [Yocto Quick Start Guide]
* [Bitbake User Manual]
* [Yocto Mega-Manual]

[Yocto Quick Start Guide]: https://www.yoctoproject.org/docs/latest/yocto-project-qs/yocto-project-qs.html

[Bitbake User Manual]: https://www.yoctoproject.org/docs/2.5/bitbake-user-manual/bitbake-user-manual.html

[Yocto Mega-Manual]: https://www.yoctoproject.org/docs/2.5/mega-manual/mega-manual.html


# Build Environment #

The nature and configuration of your build machine(s) will vary depending your
needs. This guide DOES NOT cover a configuration suitable for production build
deployments. Instead, this covers a "personal dev setup" installed onto a 
portable SSD that can be moved from machine to machine.

My minimum recommended setup is:

* (X|K)Ubuntu 16.04 or later (I use XUbuntu 17.10 and it works great.)
* 4 GiB of RAM
* Intel i5
* Over 200 GiB of disk space, preferably on an SSD

To enable me to move from machine to machine (e.g., desktop to laptop) 
with my build environment, I use a portable SSD with the following restrictions,

* The SSD is formatted with EXT4
* All machines mount the SSD to the same exact mountpoint, i.e. `/portable`
* The User and Group IDs I use between machines match. If you can't guarantee
	this with already existing users, consider making a new uid and gid for
	your builds.

If you deviate from those restrictions, you'll probably run into problems.


# Dependencies #

You'll need to install some tools to get started:

~~~
 $ sudo apt install build-essential chrpath gawk git texinfo
~~~

# Build System and Required Layers #

So, let's assume the portable SSD I'm working with is always mounted to a
/portable mountpoint.

Oftentimes I'll keep multiple Yocto releases on my system, so let's create 
a directory for the current version:

~~~
 $ mkdir -p /portable/yocto/sumo
 $ cd /portable/yocto/sumo
~~~

Let's also make some directories we'll use later:

~~~
 $ mkdir /portable/yocto/sumo/builds
 $ mkdir /portable/yocto/sumo/downloads
~~~

Now lets check out the sumo branch of "Poky" - Yocto's Build system:

~~~
 $ git clone -b sumo git://git.yoctoproject.org/poky.git
~~~

The bitbake tool used by Yocto parses "recipes" that describe how to fetch,
patch, configure, compile, install, and package software. Collections of related
"recipes" are grouped into "layers" and their names are typically prefixed 
with "*meta-"*.

There's a layer dedicated to Raspberry Pi support. Let's fetch that. Note that
we're still in /portable/yocto/sumo still!

~~~
 $ git clone -b sumo https://github.com/agherzan/meta-raspberrypi.git
~~~

If you take a look at `meta-raspberrypi/README.md`, you'll note that it has
dependencies on *meta-oe*, *meta-multimedia*, *meta-networking*, and *meta-python*.
Fortunately, these are all in one repo:

~~~
 $ git clone -b sumo git://git.openembedded.org/meta-openembedded
~~~

# Enter Your Build Environment #

Any time you want to work with the Yocto build system, you'll need your
environment to be in the proper state. A provided script sets everything up for
you.

The argument to this script is the build directory you want to work out of. This
will be created and initialized if it doesn't exist. Lets start a new build
directory called "rpi0":

~~~
 $ source /portable/yocto/sumo/poki/oe-init-build-env /portable/yocto/sumo/builds/rpi0
~~~

You should see some text welcoming you into the environment. You will be placed
into the rpi0 directory that was just created.

There are two files to configure in this directory:

* `conf/local.conf` - Build variables and options
* `conf/bblayers.conf` - Points to the layers you downloaded earlier


# Edit conf/local.conf #

Open up `conf/local.conf` in your favorite editor and find this line:

~~~
MACHINE ??= "qemux86"
~~~

Change this line to one of the following, depending on whether you have the
Raspberry Pi Zero or the Raspberry Pi Zero W:

~~~
MACHINE = "raspberrypi0"
# ... or ...
MACHINE = "raspberrypi0-wifi"
~~~

Next, change the `DL_DIR` variable to point to a common location on your
external SSD.  Locate the line that contains `DL_DIR` and change it to read as
follows:

~~~
DL_DIR = "/portable/yocto/sumo/downloads"
~~~

This variable specifies where we want to store downloaded files. For my own
sanity, I generally have separate per-platform build directories. However, to
save some time, I share the downloads between those builds so that I don't have
to re-download items I already have. This is purely a matter of preference.


# Edit conf/bblayers.conf #


Now edit `conf/bblayers.conf`. We use this file to tell Yocto (well, technically
bitbake...) where to find layers.

Use full paths here and add the paths to *meta-raspberrypi*, *meta-oe*,
*meta-multimedia*, *meta-networking*, *meta-python*. Below is a sample of what your
file will look like:

~~~
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /portable/yocto/sumo/poky/meta \
  /portable/yocto/sumo/poky/meta-poky \
  /portable/yocto/sumo/poky/meta-yocto-bsp \
  /portable/yocto/sumo/meta-raspberrypi \
  /portable/yocto/sumo/meta-openembedded/meta-oe \
  /portable/yocto/sumo/meta-openembedded/meta-python \
  /portable/yocto/sumo/meta-openembedded/meta-networking \
  /portable/yocto/sumo/meta-openembedded/meta-multimedia \
  /portable/yocto/sumo/meta-openembedded/meta-filesystems \
  "
~~~


# Build core-image-minimal #

First, let's build a minimalist Linux image called "core-image-minimal". This
will boot and drop you into a busybox environment as root. It won't include
much beyond that.

~~~
 $ bitbake core-image-minimal
~~~


# Copy core-image-minimal to an SD card #

When the build completes, an image that's ready to be dd'd to an SD card will
be in ./tmp/deploy/images

Insert an SD card and write the SD card image to it:

~~~
$ sudo dd if=./tmp/deploy/images/core-image-minimal-raspberrypi0-wifi.rpi-sdimg of=/dev/sdX bs=4M && sync
~~~


# Enable the RPi UART # 

Now before you go booting the SD card, we need to make one quick change
that we'll automate later. 

Remount or reinsert the SD card and note the presence of a `config.txt` on a FAT32
volume. This file is specfic to the Raspberry Pi boot process and is not indicative
of what is done on other platforms. Add the following line to `config.txt`:

~~~
enable_uart=1
~~~

Save this file and unmount the SD card.

# Boot core-image-minimal #

Attach a USB UART adapter to the UART pins on the raspberry pi. Here's
a terrible ASCII art diagram. Maybe just Google the pinout instead...

~~~
 .-----------------------------      --------------.
 | +5  +5  GND TX  RX                              |
 | 2                                             40|
 |  o   o   x   x   x   o         . . .      o  o  |
 |  o   o   o   o   o   o                    o  o  |
 | 1                                             39|
 |                                                 |
                                  . . .
 |                                                 |
 `--|    |----------------------     --|  |--|  |--` 
     HDMI   		               USB   USB
~~~

Plug in in the SD card and power on. You should see boot text and then a login 
prompt akin to the following:

~~~
 Poky (Yocto Project Reference Distro) 2.5 raspberrypi0-wifi /dev/ttyAMA0

 raspberrypi0-wifi login: 
~~~

Enter "root" and you'll be logged in. Look in your local.conf for an
"EXTRA_IMAGE_FEATURES" to understand why password-less root login is enabled.


# Boot rpi-hwup-image #

You may notice that lots of Raspberry Pi-specific drivers are missing. That's
because core-image-minimal did not include the platform-specific kernel modules.
The following image recipe adds these to the core-image-minimal build:

~~~
meta-raspberrypi/recipes-core/images/rpi-hwup-image.bb 
~~~

To build this, run:

~~~
$ bitbake rpi-hwup-image
~~~

When that's done, look for the `rpi-hwup-image-raspberrypi0-wifi.rpi-sdimg` in
`./tmp/deploy/images/raspberrypi0-wifi/`

Copy this to an SD card, enable the UART, and boot it.


# Making your own layer #

Having to manually edit the SD card to add that change to `config.txt` is
annoying, so let's use this as an opportunity to learn how to make our own
layer and automate this addition to our image.

Let's name our layer "meta-mypi":

~~~
$ bitbake-layers add-layer /portable/yocto/sumo/meta-mypi
~~~

Accept the default value for the "layer priority" when prompted.  Select "n"
for the example recipes -- let's not do too many things at once here. Go back
and create a separate layer to work through those later. :)

When done, you should now have a  /portable/yocto/sumo/meta-mypi directory, 
complete with a conf/ directory, an MIT license, and a boilerplate README.

For more info about creating layers, see [this section in the Yocto Project Development Tasks Manual](https://www.yoctoproject.org/docs/2.5/dev-manual/dev-manual.html#using-bbappend-files).

# Creating a rpi-config_git.bbappend #

Recipes are stored in files with a .bb extension.

If we want to add "stuff" or otherwise override definitions
in an existing recipe, we can do so by creating a corresponding .bbappend file.

A .bbappend file should have the same name as its .bb counterpart, and should
be located in the same relative path within the layer.

The recipe responsible for creating the `config.txt` file in the Raspberry Pi
image is [rpi-config_git.bb](https://github.com/agherzan/meta-raspberrypi/blob/sumo/recipes-bsp/bootfiles/rpi-config_git.bb).

In particular, the `do_deploy()` function queries variables, which you can set in
your local.conf file, and uses them to toggle various features in a `config.txt`
pulled from `github.com/Evilpaul/RPi-config.git`.

What we'll do is add a function that gets run after the `do_deploy()` and appends
out items to the end of the file.

First, let's create the proper directory structure:

~~~
 $ cd /portable/yocto/sumo/meta-mypi
 $ mkdir -p recipes-bsp/bootfiles
 $ cd recipes-bsp/bootfiles
~~~

Next, create a `rpi-config_git.bbappend` file and place the following in it:

~~~
do_deploy_append() {
    echo ""              >> ${DEPLOYDIR}/bcm2835-bootfiles/config.txt
    echo "enable_uart=1" >> ${DEPLOYDIR}/bcm2835-bootfiles/config.txt
}
~~~

That's it, as far as the bbappend goes. The last thing to do is enable your new layer in your `bblayers.conf`.

For more info about recipes and bbappends, see the following sections in the Yocto Project Development Tasks Manual:

* [Writing a New Recipe](https://www.yoctoproject.org/docs/2.5/dev-manual/dev-manual.html#new-recipe-writing-a-new-recipe)
* [Using .bbappend Files in Your Layer](https://www.yoctoproject.org/docs/2.5/dev-manual/dev-manual.html#using-bbappend-files)


# Build with the new layer enabled #

To make use of our bbappend, we have to make our build environment aware of it.
Edit `conf/bblayers.conf` and append the following to the `BBLAYERS` definition

~~~
	/portable/yocto/sumo/meta-mypi \
~~~

Now we can build and our changes will take effect:

~~~
$ bitbake hwup-rpi-image
~~~


When this completes, the SD card image should be updated to contain the new
`config.txt` file. All successive builds will include your changes to this file.

Note that you can check on the state of the `config.txt` before you spend time
writing the SD card image. During the build process, this file is written to:

~~~
./tmp/deploy/images/raspberrypi0-wifi/bcm2835-bootfiles/config.txt
~~~


# Final words #

Well, that covers about < 1% of Yocto, but it's a great start on some commodity
hardware.  Having done this, you'll hopefully now have a better sense of what's
discussed in the documentation.

If you ever get yourself into trouble, or find that a particular recipe is in a bad state,
check out the `bitbake cleansstate <recipe>` and `bitbake cleanall <recipe>` commands.

I'll leave it to you to look those up in the reference manual.

Happy hacking!
