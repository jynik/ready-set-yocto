# Ready, Set, Yocto! #

***A short, unofficial guide on getting started with Yocto Release 3.0 ("Zeus") with a Raspberry Pi***

**Release**: 3.0 "Zeus"

**Author**: Jon Szymaniak ([jynik] / [@sz_jynik])

**License**: [CC BY-SA 2.0](https://creativecommons.org/licenses/by-sa/2.0/)

*Please report bugs, typos, feedback, and violations of best practice on [the issue tracker](https://github.com/jynik/ready-set-yocto/issues).*

[jynik]: <https://github.com/jynik>
[@sz_jynik]: <https://twitter.com/sz_jynik>

# Introduction #

If you're looking to build Linux-based firmware for embedded platforms, Yocto
is a good option for you. 

If you're looking to create reproducible and version-controlled automated 
firmware builds with excellent traceability, change tracking, license auditing,
configurable QA tests, and support for tons of software... Yocto is a great option for you!

So what is Yocto exactly? It's a bit of an umbrella project that pulls together
multiple components, including the [OpenEmbedded] build system, to allow you to
create custom-tailored embedded Linux "distributions" for your specific needs.

The purpose of this guide is to get you up and running with a minimal build for
a Raspberry Pi so that you can get an initial sense of things before you dive
further into the official documentation. (There's a lot, but it's really, really well done.)

Keep the following at an arm's length. In particular, the "Mega-Manual" is fantastic
to search through (e.g. via "Ctrl-F") any time you encounter a term or variable
name that you are unfamiliar with.

* [Yocto Quick Start Guide]
* [Bitbake User Manual]
* [Yocto Mega-Manual]

## Shameless Plug ##

The follow white paper, written by yours truly, may also be
of interest to you once you start to get your bearings. I hope to update this
for the Zeus (or a later) release in the near future, so feedback on that is
always welcome.

[Improving Your Embedded Linux Security Posture with Yocto] ([archive.org mirror])

## Don't Panic! ##

Finally - please understand that there is a non-trivial learning curve with the
Yocto and OpenEmbedded projects. Don't be discouraged, and reach out if you need help.
Both projects have mailing lists, as well as a presence on sites like Stack Exchange.
I generally will try to provide advice if folks reach out to me directly, time
and general life situation permitting.

If you ultimately find that you're having a lot of trouble or just don't like something
about Yocto/OpenEmbedded, consider kicking the tires on BuildRoot. You can find
comparisons of all three projects [here](https://www.yoctoproject.org/learn-items/elc-2016-buildroot-vs-yocto-projectopenembedded-alexandre-belloni/).

[Yocto Quick Start Guide]: https://www.yoctoproject.org/docs/latest/yocto-project-qs/yocto-project-qs.html

[Bitbake User Manual]: https://www.yoctoproject.org/docs/3.0/bitbake-user-manual/bitbake-user-manual.html

[Yocto Mega-Manual]: https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html

[OpenEmbedded]: https://www.openembedded.org/wiki/Main_Page

[Improving Your Embedded Linux Security Posture with Yocto]: https://www.nccgroup.com/globalassets/our-research/us/whitepapers/2018/improving-embedded-linux-security-yocto3.pdf

[archive.org mirror]: https://web.archive.org/web/20220122003633/https://www.nccgroup.com/globalassets/our-research/us/whitepapers/2018/improving-embedded-linux-security-yocto3.pdf


# Build Environment #

The nature and configuration of your build machine(s) will vary depending your
needs. This guide DOES NOT cover a configuration suitable for production build
deployments. Instead, this covers a "personal dev setup" installed onto a 
portable SSD that can be moved from machine to machine.

My (minimum) recommended setup is:

* (X|K)Ubuntu 18.04 LTS (or some similarly recent-ish distro of your choice)
* At least 8 GiB of RAM
* At least an Intel i5 (or AMD equivalent) 
* At least 200 GiB of disk space, preferably on an SSD

To enable me to move from machine to machine (e.g., desktop to laptop) 
with my build environment, I use a portable SSD with the following restrictions,

* The SSD is formatted with EXT4
* All machines mount the SSD to the same exact mountpoint, i.e. `/portable`
* The User and Group IDs I use between machines match. If you can't guarantee
	this with already existing users, consider making a new uid and gid for
	your builds.

If you decide to use a removable volume that's shared between multiple machines
and deviate from those restrictions, you'll probably run into problems.


# Dependencies #

You'll need to install some tools to get started. For the apt users out there...

~~~
 $ sudo apt install build-essential chrpath gawk git texinfo
~~~

# Build System and Required Layers #

So, let's assume the portable SSD I'm working with is always mounted to a
`/portable` mountpoint.

Oftentimes I'll keep multiple Yocto releases on my system, so let's create 
a directory for the current version:

~~~
 $ mkdir -p /portable/yocto/zeus
 $ cd /portable/yocto/zeus
~~~

Let's also make some directories we'll use later:

~~~
 $ mkdir /portable/yocto/zeus/builds
 $ mkdir /portable/yocto/zeus/downloads
~~~

Now lets check out the zeus branch of "Poky" - Yocto's Build system:

~~~
 $ git clone -b zeus git://git.yoctoproject.org/poky.git
~~~

The bitbake tool used by Yocto parses "recipes" that describe how to fetch,
patch, configure, compile, install, and package software. Collections of related
"recipes" are grouped into "layers" and their names are typically prefixed 
with "*meta-"*.

There's a layer dedicated to Raspberry Pi support. Let's fetch that. Note that
we're still in `/portable/yocto/zeus` still!

~~~
 $ git clone -b zeus https://github.com/agherzan/meta-raspberrypi.git
~~~

If you take a look at `meta-raspberrypi/README.md`, you'll note that it has
dependencies on *meta-oe*, *meta-multimedia*, *meta-networking*, and *meta-python*.
Fortunately, these are all in one repo:

~~~
 $ git clone -b zeus git://git.openembedded.org/meta-openembedded
~~~

# Enter Your Build Environment #

Any time you want to work with the Yocto build system, you'll need your
environment to be in the proper state. A provided script sets everything up for
you.

The argument to this script is the build directory you want to work out of. This
will be created and initialized if it doesn't exist. Lets start a new build
directory called "rpi":

~~~
 $ source /portable/yocto/zeus/poky/oe-init-build-env /portable/yocto/zeus/builds/rpi
~~~

You should see some text welcoming you into the environment. You will be placed
into the *rpi* directory that was just created.

There are two files to configure in this directory:

* `conf/local.conf` - Build variables and options
* `conf/bblayers.conf` - Points the build environment to the layers you want to use


# Edit conf/local.conf #

Open up `conf/local.conf` in your favorite editor and find this line:

~~~
MACHINE ??= "qemux86"
~~~

This denotes which platform we're targeting. This string corresponds
to `<MACHINE>.conf` files traditionally stored in the `conf/machine/` directories
provided by various "layers". 

## A quick aside regarding MACHINE... #

Take a look through the contents of the `meta-raspberrypi/machine/` directory. Observe
that there's a `.conf` file for each revision of the Raspberry Pi, and in some cases,
different variants of the same revision. 

For instance, there's a "`raspberrypi0`" and
a "`raspberrypi0-wifi`" -- the former is the variant without built-in Wi-Fi, and the
later targets the variant with is. The "`raspberrypi4-64`" configuration operates
the RPi 4 in its 64-bit mode, while the "`raspberrypi4`" machine configuration
operates it in 32-bit mode.  

If you look at the content in these machine configuration files, you'll find that
there's not a whole lot in there. This is because a majority of the "common stuff"
is all tucked away in files that are included (*think #include in C/C++*) via the
[`include`](https://www.yoctoproject.org/docs/3.0/bitbake-user-manual/bitbake-user-manual.html#include-directive) and [`required`](https://www.yoctoproject.org/docs/3.0/bitbake-user-manual/bitbake-user-manual.html#require-inclusion) directives.

Thus, the machine configuration files only override or expand upon variable definitions as-needed to do things like...

* Specify the U-Boot or Linux kernel target platform name
* Append extra items to a list of recommended firmware (e.g., Wi-Fi, BlueTooth module) recipes
* Tweak kernel command line arguments (e.g. serial console)

## Back to local.conf ##

So now that we know a little bit more about what the "`MACHINE`" variable does,
select the one that fits that platform that you'd like to build an image for.

As an example, let's target the Raspberry Pi 3 in 32-bit mode... just because we can.
This is done by changing the `MACHINE` definition in `local.conf` to:

~~~
MACHINE = "raspberrypi3"
~~~

Next, change the `DL_DIR` variable to point to a location where we should
store downloaded files. I recommend setting this to something like...

~~~
DL_DIR = "/portable/yocto/zeus/downloads"
~~~

This variable specifies where we want to store downloaded files. For my own
sanity, I generally have separate per-platform build directories. However, to
save some time, I share the downloads between those builds so that I don't have
to re-download items I already have. This is purely a matter of preference.

## Raspberry Pi-Specific Configurations ##

Before we're done editing the `local.conf` file, take a quick skim through the
`meta-raspberrypi` layer's [`docs/extra-build-config.md`](https://git.yoctoproject.org/cgit/cgit.cgi/meta-raspberrypi/tree/docs/extra-build-config.md?h=zeus) file.  

This describes a variety of `local.conf` definitions that you can use to
enable/disable/modify what ultimately gets written to the `config.txt` file
stored in the image's boot volume, which used to [configure the Raspberry Pi at
runtime](https://www.raspberrypi.org/documentation/configuration/config-txt/README.md).

Frankly, the Raspberry Pi is bit weird and atypical in this regard; this type
of configuration deviates a bit from the spirit of Yocto/OpenEmbedded, which I
suspect stems from the fact that it's a designed to be a platform for people
modify, customize, and hack on -- this differs from a product designed to implement
only a very specific set of requirements, which is what the build system is
intended for.

If you plan on distributing your custom firmware build setup to other people
(or are responsible for maintaining the firmware build process for your
product), avoid heavy use of local.conf tweaks. Instead use custom layers, custom
machine configuration files, and  "bbappend" files -- we'll learn about those more a bit later.  

For now, just roll with this, and know there's a "better" approach for builds
that aren't just one-off experiments.

For the Raspberry Pi, below two settings you'll want to consider enabling, via
the documented additions to your `local.conf`. Note that there's a few more
you may be interested in if you're planning to use I2C and SPI interfaces.

* `ENABLE_UART = "1"`    - If you'd like to access a shell via the UART
* `RPI_USE_U_BOOT = "1"` - If you'd like to use the U-Boot bootloader


# Edit conf/bblayers.conf #

So far we've mentioned "layers" a few times, but haven't discussed how our
build environment knows which ones to use. That's what we're going to do now!

Briefly examine the contents of the `conf/bblayers.conf` file. Note that the
only entries in the `BBLAYERS` list are those associated with `poky`. We 
need to add the full paths to the `meta-rasperrypi` and subdirectories within
the `meta-openembedded` repository we cloned earlier.

Although we could edit this file manually, there's a `bitbake-layers` command
that will help us out. Run the following command to see information about its
usage:

~~~
$ bitbake-layers -h
~~~

From the output of this command, we can see that we want to run the `add-layer`
sub-command. Run the command shown below, which has been split onto multiple
lines here for readability:

~~~
$ bitbake-layers add-layer \
	/portable/yocto/zeus/meta-openembedded/meta-oe \
 	/portable/yocto/zeus/meta-openembedded/meta-multimedia \
 	/portable/yocto/zeus/meta-openembedded/meta-networking \
 	/portable/yocto/zeus/meta-openembedded/meta-python \
 	/portable/yocto/zeus/meta-raspberrypi 
~~~

Be sure to use full paths here. Bitbake may support otherwise nowadays, but
I personally prefer to stick with full paths lest I forget any important caveats.

After this command completes, note that the `bblayers.conf` file has been updated.
Below is a sample of what you'll see.

~~~
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /portable/yocto/zeus/poky/meta \
  /portable/yocto/zeus/poky/meta-poky \
  /portable/yocto/zeus/poky/meta-yocto-bsp \
  /portable/yocto/zeus/meta-openembedded/meta-oe \
  /portable/yocto/zeus/meta-openembedded/meta-multimedia \
  /portable/yocto/zeus/meta-openembedded/meta-networking \
  /portable/yocto/zeus/meta-openembedded/meta-python \
  /portable/yocto/zeus/meta-raspberrypi \
  "
~~~

# Build core-image-minimal #

Let's start by building a very minimalist Linux image called
"core-image-minimal". This will boot and drop you into a busybox environment as
root. It won't include much more beyond that, however.

~~~
 $ bitbake core-image-minimal
~~~


# Copy core-image-minimal to an SD card #

When the build completes, an image that's ready to be dd'd to an SD card will
be in `./tmp/deploy/images`. (That's a directory named `tmp` within your build directory, **not** `/tmp`.)

Insert an SD card and write the SD card image to it, replacing `sdX` with the
appropriate block device. 

*Take care not to clobber data on the wrong drive due to a typo at this point!*

~~~
$ sudo dd if=./tmp/deploy/images/raspberrypi3/core-image-minimal-raspberrypi3.rpi-sdimg of=/dev/sdX bs=4M
$ sync
~~~

# Boot core-image-minimal #

Attach a USB-UART adapter to the UART pins on the Raspberry Pi's expansion
connector as follows:

~~~
GND: Pin 6
TX:  Pin 8
RX:  Pin 10
~~~

Open the UART up in minicom, screen, etc. Plug in in the SD card and power on. 

Via the UART, you should see boot text and then a login prompt akin to the
following:

~~~
 Poky (Yocto Project Reference Distro) 3.0 raspberrypi3 /dev/ttyS0

 raspberrypi3 login: 
~~~

Enter "root" and you'll be logged in. Look in your local.conf for an
"EXTRA_IMAGE_FEATURES" to understand why password-less root login is enabled,
let alone why root login is even enabled. Hint - it's a debug feature that you
have to *opt-out* of!


# Building a more functional base image #

From the console on the Raspberry Pi, run `lsmod`.  If you've used full-featured
and general purpose distributions like Raspbian in the past, you might notice that
none of the drivers you might have expected to see are loaded!

That's because core-image-minimal did not include the platform-specific kernel
modules; it's a minimalist base image that you can expand upon to add additional
functionality.

In fact, the `core-image-base` recipe does just that, and may very well be a better
starting point for your custom firmware images!

To build this, run:

~~~
$ bitbake core-image-base
~~~

You'll see a bunch of new software being fetched, compiled, and staged for
deployment in the firmware image. However, everything you've already built
(that doesn't require rebuilding due to a configuration change) remains
cached -- each successive build you do will generally require far less time.

When that's done, look for the `core-image-base-raspberrypi3.rpi-sdimg` in
`./tmp/deploy/images/raspberrypi3/` .

Copy this to an SD card as you did earlier and boot it.  Run `lsmod` again and
observe that a whole slew of drivers have now been included in the image. Note 
that the HDMI interface is operational, along with WLAN and BlueTooth interfaces.

# Creating your own layer #

At some point soon, you'll probably want to integrate your own software,
scripts, or configuration changes into an image. Creating your own custom
layer is the right place to begin introducing those changes. 

As a general rule of thumb, you shouldn't be modifying the contents of other
layers, provided that they are maintained reasonably well. Even in the cases
when they aren't, there are often features (like [BBMASK](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#var-BBMASK)) that you can use to deal with undesired behavior from
other layers, without needing to worry about maintaining a fork of the
problematic layer.

Let's start with an overkill, but simple example, just to go through the
process of creating a new layer. Let's say you've added some initscripts to an
image, and after some debugging you've realized that BusyBox isn't configured
with a particular command or argument that you need. 

For example, try to run the commands "xz" and "unxz" from your shell on the
Raspberry Pi and observe that they're not present on the system. (At the time of
writing, these file compression Busybox "applets" are not enabled by default.)
So, we'll enable these!

Copying entire recipes into your layer is considered a [poor practice](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#best-practices-to-follow-when-creating-layers). Instead,
the Yocto and OpenEmbedded communities prefer the use of a
"[*bbappend*](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#using-bbappend-files)"
file, which will effectively let you *append* to the definition of a recipe
found elsewhere. This allows you modify or redefine variable definitions, as
well as make changes to the way software is fetched, compiled, "installed", and
deployed.

In our case, we're going to use a *bbappend* file to change the `.config` file
settings used to compile BusyBox.

First, we need to create a new layer. Let's call it "meta-mypi" and create it
using that `bitbake-layer` command we used earlier:

Let's name our layer "meta-mypi":

~~~
$ bitbake-layers create-layer /portable/yocto/zeus/meta-mypi
~~~

When done, you should now have a  /portable/yocto/zeus/meta-mypi directory,
complete with a conf/ directory, an MIT license, a boilerplate README, and
an example recipe that we won't concern ourselves with here.

For more info about creating layers, see [this section in the Yocto Project Development Tasks Manual](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#understanding-and-creating-layers).

# Creating a bbappend file for BusyBox #

## First, a bit of background ##

Recipes are stored in files with a .bb extension.

If we want to add "stuff" or otherwise override definitions in an existing
recipe, we can do so by creating a corresponding .bbappend file in our own layer.

A .bbappend file must have the same name as its .bb counterpart, and as a
matter of best practice, should be located in the same relative path within the
layer.

The BusyBox recipe is found in [`poky/meta/recipes-core/busybox/busybox_1.31.0.bb`](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/recipes-core/busybox/busybox_1.31.0.bb?h=zeus). 
(The exact version string may have change since this tutorial was last updated.)

Note that this bitbake recipe includes the [`busybox.inc`](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/recipes-core/busybox/busybox.inc?h=zeus) file (via a `require` directive),
where most of the work of the recipe is actually implemented. 

The [`poky/meta/recipes-core/busybox/busybox`](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/recipes-core/busybox/busybox?h=zeus) directory contains patches, "configuration
fragments", while [`poky/meta/recipes-core/busybox/files`](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/recipes-core/busybox/files?h=zeus) contains items to deploy within the target filesystem image. You'll see these referenced via "file://" in the recipe's [`SRC_URI`](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#var-SRC_URI) definition.

The term "[configuration
fragments](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#creating-config-fragments)"
is most often used with respect to Linux kernel configuration, but applies to
the KConfig-esque configuration of BusyBox as well. Basically, these are `.cfg`
files that contain just a handful of `CONFIG_FEATURE_X=y` entries. For recipes
designed to handle this this KConfig-like convention, we can simply use our bbappend
to introduce a `.cfg` file that modifies our desired configuration tweak.

*(Another quick aside...*)

We won't talk about it here, and don't get bogged down by it, but know that
when using bbappend to tweak recipes (most of which don't use config fragments)
you may find yourself needing to add a little extra bit of shell or Python code
in a function within a recipe.  Keep an eye out (i.e. grep around layers) for
recipes and bbappend files defining functions named `do_configure_append()` or
`do_configure_prepend()` to see examples of this. The Bitbake manual section
about [Functions](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#functions)
explains what's going on with this syntax.

## Identifying what we want to change ##

Okay, so we want to enable "xz" and "unxz", but where do we start?

Let's figure out what the state of the `.config` file currently is, for the
Busybox binary we've already built.

To do this, we can do a few things, including:

1. Inspect the [`poky/meta/recipes-core/busybox/busybox/defconfig`](https://git.yoctoproject.org/cgit/cgit.cgi/poky/tree/meta/recipes-core/busybox/busybox/defconfig?h=zeus) file, which
    gets combined with configuration fragments at build time.

2. Look at the actual .config file that was used to configure the build that has already happened.


For the sake of learning more about bitbake, let's do number 2. Run the following command.
This will launch a "devshell" which puts you in a fakeroot-esque environment, located at the
working directory used when processing the BusyBox recipe.

~~~
$ bitbake -c devshell busybox
~~~

Once you're in this environment, run the command `pwd` to see where exactly bitbake has plopped us.
You'll find that we're down a few directories into `./tmp/work/`.

While in the directory where our devshell has placed us, open the `.config` file with your favorite editor.
Search for "XZ" and note that `CONFIG_XZ` and `CONFIG_UNXZ` are unset -- these are what we want to enable.

## Implementing our change via a bbappend and a config fragment ##

Close the console opened for the dev shell and navigate to your "meta-mypi"
directory in a different terminal from the one you're using to run bitbake
commands. 

Create a `meta-mypi/recipes-core/busybox/busybox` directory, just to mirror the paths we saw earlier
in the `poky/meta` directory tree. Then, create a configuration fragment file in it that specifies
that we want `CONFIG_XZ=y` and `CONFIG_UNXZ=y`.

~~~
$ mkdir -p /portable/yocto/zeus/meta-mypi/recipes-core/busybox/busybox
$ echo "CONFIG_XZ=y"    > /portable/yocto/zeus/meta-mypi/recipes-core/busybox/busybox/xz.cfg
$ echo "CONFIG_UNXZ=y" >> /portable/yocto/zeus/meta-mypi/recipes-core/busybox/busybox/xz.cfg
~~~

Next, lets create our actual bbappend file. Create the following file, again adjusting
slightly if the busybox version has changed since the time of writing.

~~~
/portable/yocto/zeus/meta-mypi/recipes-core/busybox/busybox_1.31.0.bbappend
~~~

You only need to put two lines in this file. Ensure you include the leading
whitespace in the string on the second line.

~~~
FILESEXTRSPATHS_prepend := "${THISDIR}/${PN}:"
SRC_URI_append := " file://xz.cfg"
~~~

By setting the [`FILESEXTRASPATHS`](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#var-FILESEXTRAPATHS) variable, we're instructing the OpenEmbedded build system to look inside the directory in which our bbappend file resides (`${THISDIR}`), in a a subdirectory with the same name as our recipe, busybox. (Refer to the [glossary, regarding `${PN}`](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#var-PN)).

That's it, as far as the bbappend goes. The last thing to do is enable our new layer, since we haven't done that yet.

One final note - it would be incredibly annoying if you had to create a new bbappend file for every patch, or even minor, revision bump to a piece of software. As noted [here](https://www.yoctoproject.org/docs/3.0/mega-manual/mega-manual.html#append-bbappend-files), you can actually use a "%" wildcard in the version portion of a recipe filename to match multiple version. We could have named our file, `busybox_1.31.%.bbappend`, for example, to match all versions in teh 1.31.x series.

## Enable our layer ##

From the terminal where you've been running `bitbake` commands, run `bitbake-layers add-layer` to add our "meta-mypi" layer. Refer back to the earlier section if you've forgotton how to do this.

Double check your `conf/bblayers.conf`, just to see that the change took effect.

## Build with the new layer enabled ##

Now we can build our firmware image and our changes will take effect. First, we'll issue
a "cleansstate" command to ensure busybox is reconfigured and rebuilt.

~~~
$ bitbake -c cleansstate busybox
$ bitbake core-image-base
~~~

Technically, that first command isn't necessary -- bitbake will detect the change and will
identify all the `core-image-base` dependencies that require rebuilding. It's quite smart like that.

However, I like to be explicit, just for my own sanity.  *Plus, this gave me the chance to note how to
fully "clean" up the state of a recipe build, without having to have it
redownloaded, which is what happens if you use "*cleanall*" instead of
"*cleansstate*".  Using "*clean*" is essentially like running "make clean" --
it will clean up compilation, but not configuration, artifacts.*

When this completes, run `bitbake devshell busybox` again and confirm that the `.config`
file has indeed been updated to reflect our changes. Then write the newly regenerated 
`core-image-base-raspberrypi3.rpi-sdimg` to an SD card, boot it, and observe that you can now
use the "xz" and "unxz" commands in all their glory.


# Final words #

Well, that covers about < 1% of Yocto, but it's a great start on some commodity
hardware.  Having done this, you'll hopefully now have a better sense of what's
discussed in the documentation.

If you ever get yourself into trouble, or find that a particular recipe is in a bad state,
check out the `bitbake cleansstate <recipe>` and `bitbake cleanall <recipe>` commands.

I'll leave it to you to look those up in the reference manual.

There's definitely quite a learning curve here, but I've found that the best way to tackle it is to
just explore existing recipes, along with the implementation of the Yocto and OpenEmbedded components themselves.

If you run into strange errors, always start by checking for variable name typos. Those will getcha! If you find yourself questioning your sanity, `bitbake -e` is your friend. I'll let you look that one up too.

<br>
*Good luck and happy hacking!*

*- jynik*
