# Getting started with gst-build on a debian based machine

## Introduction

Seen the confusion it can result using our favorite framework named GStreamer. I will
try to describe the way i'm working with this multiple repositories configuration.
First you can of course retrieve the repository you'd like to work on, by example
I want to finetune a plugin in gst-plugins-good, so I retrieve gst-plugins-good
from gitlab.freedesktop.org and try to build with the right dev package from my system.

But that's where it can start to be complicated:
- wrong version of base gstreamer libs
- missing dependencies to build
- etc.

In order to solve the issue, two build tools are available to help users build their
own distribution of GStreamer:

 - cerbero
 - gst-build
 

In the blog post I will try to describe the two with their advantages and their drawback

## Environment

Ubuntu 18.04 with a regular terminal

## Cerbero

### Description

This tool has been implemented initially by Andoni Morales(ylatuya) in order to address a way to package
the gstreamer libraries and plugins. 
This build tool will build the maximum to create a stable sandbox
avoiding unexpected behavious from distribution package. By example glib which is the main library used
by GStreamer is using a given version in cerbero and not the one from your system.
This tool is written in python and work on al the platform such as Linux, MacOS, Windows. It can build for
these platforms plus other dedicated platform such as Android, IOS etc.

### Advantages

 * Easy to add new platform
 * Easy python code
 * Good sandbox with easy recipe system
 * Support the modern system such as
 * Can package for all platforms (Linux, MacOS, Windows, IOS, Android)
 * support static build
 * Support cross build
 
### Drawbacks

 * difficult to use when developping
 * slow and footprint

### How to install

#### Prerequisites

 * python3
 * git
 
#### Get the repository

I assume your working directory will be `/opt`

```
# git clone https://gitlab.freedesktop.org:gstreamer/cerbero.git
```

#### bootstrap

This command will install distributions package you'll need on your system. This is totally transparent.
On Debian based system you just have to confirm and then it will apt all the necessary package.

This command is mandatory before starting to build anything

```
# cd cerbero
# ./cerbero-uninstalled bootstrap
```

#### build gst-plugins-good

```
# ./cerbero-uninstalled build gst-plugins-good-1.0
```

The build artifacts can be found `cerbero_path/build/sources/linux_x86_64/gst-plugins-good-1.0-1.17.0.1` and
the build logs can be found in `cerbero_path/build/logs/linux_x86_64/gst-plugins-good-1.0-compile.log` or any
of the steps for any of the packages

#### package gst-plugins-good

```
# ./cerbero-uninstalled package gstreamer-1.0
```

The build artifacts will be a deb/rpm package and can be found `/opt/cerbero/`


#### Cross compile for Windows

```
# ./cerbero-uninstalled -c config/cross-win64.cbc bootstrap
# ./cerbero-uninstalled -c config/cross-win64.cbc package gstreamer-1.0
```

This command will generate a MSI package which can be used directly on awindows platform to install
gstreamer.

#### Development environment

If you want to modify some code in gst-plugins-good, you can use the `shell` environment and then go to the 
given folder such as `/opt/cerbero/build/sources/linux_x86_64/gst-plugins-good-1.0-1.17.0.1` but that's where
i can be a bit tricky seen that you work will be destroyed each time you gonna rebuild with cerbero.

Indeed this folder is a fetch from `$HOME_DIR/.cache/cerbero-sources/gst-plugins-good-1.0`

So either you can add your own remote and work in this folder but take care that it will be destroyed on the 
build.

I recommend to have your own repository somewhere else with your given remote and build it with
the value used by cerbero.

```
# cd DEV
# git clone https://itlab.freedesktop.org:gstreamer/gst-plugins-good.git
# cd gst-plugins-good
```
In order to get the configure line you have to edit the given file and get the first line of the file
```
# vi /opt/cerbero/build/logs/linux_x86_64/gst-plugins-good-1.0-configure.log
```
Then you can run it 
```
# /opt/cerbero/build/build-tools/bin/meson build_dir --prefix=/opt/cerbero/build/dist/linux_x86_64 --libdir=lib --default-library=both --buildtype=debugoptimized --backend=ninja --wrap-mode=nodownload --native-file /opt/cerbero/build/sources/linux_x86_64/gst-plugins-good-1.0-1.17.0.1/_builddir/meson-native-file.txt -Dcairo=enabled -Ddv=enabled -Dflac=enabled -Dgdk-pixbuf=enabled -Djpeg=enabled -Dlame=enabled -Dmpg123=enabled -Dpng=enabled -Dsoup=enabled -Dspeex=enabled -Dtaglib=enabled -Dvpx=enabled -Dwavpack=enabled -Daalib=disabled -Ddv1394=disabled -Dgtk3=disabled -Djack=disabled -Dlibcaca=disabled -Doss=disabled -Doss4=disabled -Dqt5=disabled -Dshout2=disabled -Dtwolame=disabled -Dv4l2=enabled -Dx11=enabled -Dpulse=enabled -Dexamples=disabled
```

Then you can build
```
ninja -C buid_dir
```

Finally you can install and test your mods
```
ninja -C buid_dir install
```

Then lets implement your new plugin or polish the beautiful gstreamer piece of art. 


## gst-build

### Description

This tool has been implemented a few years ago (2016) by Thibault Saunier to replace the former existing tool
`gst-uninstalled`.
But seen that GStreamer moved his build system to meson recently (and drop autotools support September 2019), 
the logical path is now to use meson to replace gst-uninstalled.
This tool has been implemented to build everything in one almost one command.

### Advantages
 * easy to install
 * fast build system
 * easy to use to develop your own plugin
 * very small footprint
 
### Drawbacks
 * not cross platform
 * a bit difficult to configure
 * rely on system dependencies


### How to install

#### Prerequisites

 * python3
 * git
 * meson
 * ninja
 
#### Install meson and ninja

Assuming you already install pip3

```
 pip3 install --user meson
``` 

This will install meson into ~/.local/bin which may or may not be included automatically in your PATH by default.

```
sudo apt install ninja
```
 
#### Get the repository

I assume your working directory will be `/opt`

```
# git clone git@gitlab.freedesktop.org:gstreamer/gst-build.git
```

#### Fetch && configure

This step will download the GStreamer repository including some dependencies such as glib etc. Mainly
it tries to download as much as possible the 3rd parties which are using meson as a build system and
breaking news the cmake one seen that a bridge has been implemted recently.

```
meson build_dir
```
##### Build gst-build

```
ninja -C build_dir
```

##### Update gst-build

This command will update all the repos and will reissue a build. Take care this command will destroy your dev
branch
```
ninja -C build_dir update
```
#### Test gst-build

This command will create a new environment where all the plugins built previously are available.
```
ninja -C build_dir uninstalled
```

From this environment you are now ready to use the power of GStreamer even implement new features in it without
the fear to not have the up2date version.

#### Modifying an existing repository

You can as described with `cerbero` go to a repo folder such as gst-plugins-good and modify/add code to the
repository but in an handier manner here.
All the repositories can be in `/opt/gst-build/subprojects`, so you can go to 
`/opt/gst-build/subprojects/gst-plugins-good` and modify the code and then come back to `/opt/gst-build` and
reuissue the build in the `uninstalled` environment. That's magic !

#### Adding a new repository

Better to be outside of `uninstalled` env.
If you want to add a new repository and work in this environment. Very simple and handy way, you'll have to:

```
cd subprojects
git clone my_subproject
cd ../build
rm -rf * && meson .. -Dcustom_subprojects=my_subproject
ninja
```

And then you can go in your subproject, edit, change, remove even stare at his beauty and then you go back to
the root of `gst-build` and issue a:

```
ninja -C build
```

## Conclusion

Not very sure who is best for sure and the other be dropped but I would say that cerbero is good tool to package
or debug an existing bug where the environment is stable with the less dependencies to the system. But for the
flexibility and the rapidity, I will advise to use gst-build which can used also in a stable environment using
a docker image by example.
