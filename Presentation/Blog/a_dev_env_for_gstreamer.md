# Getting started with gst-build on a debian based machine

## Introduction

Seen the confusion it can result using our favorite framework named GStreamer can bring about. I will
try to describe the way i'm working with this massive project spreading across multiple repositories.
First you can of course retrieve the repository you'd like to work on. For example
`I want to finetune a plugin in gst-plugins-good`, so I clone gst-plugins-good
from gitlab.freedesktop.org, install all its dependencies using -dev packages from my distro's repository and
run the build fingers crossed.

But that's where it can start to get complicated:
- wrong version of base gstreamer libs
- missing dependencies to build
- etc.

In order to solve the issue, two systems are available to help users build their
own distribution of GStreamer:

 - cerbero
 - gst-build
 

Let's have a look at the advantages and drawbacks of the two.

## Environment

 * Ubuntu 18.04
 * Bash Shell

## Cerbero

### Description

This tool was implemented initially by Andoni Morales(ylatuya) in order to address a way to package
the gstreamer libraries and plugins. 
It will build will build GStreamer and as many of its dependencies as possible from scratch,
creating a stable sandbox that protects the developer from the problems and unexpected behaviors
the unverified mix of distribution libraries may cause. For example Cerbero will use a specific tested version
of GLib, one of the major libraries GStreamer is built upon,
instead of whatever random release your system happens to provide..
Because Cerbero is written in Python, it will run on all major operating systems (GNU/Linux, MacOS, Windows),
from where it is also able to cross compile binaries for Android, iOS and a couple more embedded platforms

### Advantages

 * Convenient to add new platform
 * Easy reading python code
 * Good sandbox with easy recipe system
 * Can package GStreamer for all platforms (Linux, MacOS, Windows, iOS, Android)
 * Support static build
 * Support cross build
 
### Drawbacks

 * Difficult to use when developing beause it will wipe out your changes on the next cerbero's rebuild.
 * The build process can be slow because of heterogeneous build system used.
 * The memory footprint is important because of the sandbox

### How to install

#### Prerequisites

 * python3
 * git
 
#### Get the repository

I'll assume the working directory will be `/opt`

```
# git clone https://gitlab.freedesktop.org:gstreamer/cerbero.git
```

#### bootstrap

This command will install distribution packages you'll need on your system. This is totally transparent.
On Debian-based system you just have to confirm and then it will apt all the necessary packages.

This command is mandatory before starting to build anything

```
# cd cerbero
# ./cerbero-uninstalled bootstrap
```

#### build gst-plugins-good

```
# ./cerbero-uninstalled build gst-plugins-good-1.0
```

The build artifacts can be found `cerbero_path/build/sources/linux_x86_64/gst-plugins-good-1.0-x.x.x.x` and
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

This command will generate a MSI package which can be used directly on a Windows platform to install
gstreamer.

#### Development environment

If you want to modify some code in gst-plugins-good, you can use the `shell` environment and then go to the 
given folder such as `/opt/cerbero/build/sources/linux_x86_64/gst-plugins-good-1.0-1.17.0.1` but that's where
i can be a bit tricky seen that you work will be destroyed each time you gonna rebuild with cerbero.


#### Let's add a log line in gst-plugins-base

In this tutorial, I will explain how to add a log line in videotestsrc, gst-plugins-base's plugin, rebuild using
cerbero environment and test that the new log is now displayed.

##### Go to cerbero's shell

```
./cerbero-uninstalled shell
```

You should have a new prompt with the prefix `[cerbero-linux-x86_64]`

##### Clone the repository

Seen that cerbero might erase the change in its build folder on a next build, I will describe, here,
the way i'm using my own repository and configure, edit and build it using cerbero's environment.

```
cd /opt
git clone https://gitlab.freedesktop.org:gstreamer/gst-plugins-base.git
cd gst-plugins-base
```

##### Get cerbero's configure line

You should be able to get the meson's build line for gst-plugins-base in 
`cerbero_path/build/logs/linux_x86_64/gst-plugins-base-1.0-configure.log`

Something like:

```
/opt/cerbero/build/build-tools/bin/meson --prefix=/opt/cerbero/build/dist/linux_x86_64 --libdir=lib
--default-library=both --buildtype=debugoptimized --backend=ninja --wrap-mode=nodownload
--native-file /opt/cerbero/build/sources/linux_x86_64/gst-plugins-base-1.0-1.17.0.1/_builddir/meson-native-file.txt
-Dgl=enabled -Dlibvisual=enabled -Dogg=enabled -Dopus=enabled -Dpango=enabled -Dtheora=enabled
 -Dvorbis=enabled -Dtremor=disabled -Dcdparanoia=enabled -Dx11=enabled -Dxvideo=enabled -Dalsa=enabled
 -Dintrospection=enabled -Dexamples=disabled
```

This command will configure the build in the source folder, you change the build folder by calling

```
meson build_dir ...
```

##### Edit the file
```
vim gst/videotestsrc/gstvideotestsrc.c
```
Go to the method `gst_video_test_src_start` and edit the file by adding for example:

```
GST_ERROR_OBJECT (src, ""Starting to debug videotestsrc");
```

Then close the editor.

##### Build and  install 

This comamnd will build `gst-plugins-base` and install the artifacts in $CERBERO_PREFIX

```
ninja install
```

##### Test the changes

In order to enable the logs, you have to export the environment variable GST_LOG. You'll find further infos
visiting this [page](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c)

Let's start the playback and display the result in the terminal. The following command will display
all the log from videotestsrc with the category ERROR(1).

```
GST_DEBUG=videotestsrc:1 gst-launch-1.0 videotestsrc num-buffers=1 ! fakevideosink
```
You should have this output:

```
Setting pipeline to PAUSED ...
0:00:00.225273663 21743 0x565528ab7100 ERROR           videotestsrc gstvideotestsrc.c:1216:gst_video_test_src_start:<videotestsrc0> Starting to debug videotestsrc
Pipeline is PREROLLING ...
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
New clock: GstSystemClock
Got EOS from element "pipeline0".
Execution ended after 0:00:00.033464391
Setting pipeline to PAUSED ...
Setting pipeline to READY ...
Setting pipeline to NULL ...
Freeing pipeline ..
```

#### Implement your own plugin

I will recommend in the case to create your own recipe in cerbero for your given project. You can find examples
in the cerbero/recipes/*.recipe. Cerbero supports autotools/cmake/meson regular build system but you can
also implement your own build steps.


## gst-build

### Description

This tool was implemented a few years ago (2016) by Thibault Saunier to replace the former existing tool
`gst-uninstalled`.
But seen that GStreamer moved his build system to meson recently (and drop autotools support in September 2019), 
the logical path is now to use meson to replace `gst-uninstalled.py`.
It was implemented to build everything in almost one command.

### Advantages
 * Easy to install and use.
 * Faster build system.
 * Easy to implement your own plugin with this environment.
 
### Drawbacks
 * not cross platform
 * a bit difficult to configure
 * rely on system dependencies which can result in missing features.


### How to install

#### Prerequisites

 * build-essential (gcc)
 * python3
 * git
 * meson
 * ninja
 
#### Install meson and ninja

Assuming you already install pip3, the Python package installer.

```
 pip3 install --user meson
``` 

This will install meson into ~/.local/bin which may or may not be included automatically in your PATH by default.

```
sudo apt install ninja
```
 
#### Get the repository

I'll assume the working directory will be `/opt`

```
# git clone git@gitlab.freedesktop.org:gstreamer/gst-build.git
```

#### Fetch && configure

This step will download the GStreamer repositories including some dependencies such as glib etc. Mainly
it tries to download as much as possible the 3rd parties which are using meson as the main build system and
`breaking news` the cmake one seen that a bridge has been implemented recently.

```
meson build_dir
```

##### Build gst-build

This step will build all GStreamer base libs in addition to the plugins from base/good/bad/ugly/libav if their
depencies have been met.

```
ninja -C build_dir
```


#### Test gst-build

This command will create a new environment where all tools and plugins built previously are available.
```
ninja -C build_dir uninstalled
```
A prefix to  your prompt should be available as

```
[gst-master] bash-prompt #
```

From this environment you are now ready to use the power of GStreamer even implement new features in it without
the fear of using a non up2date version.

##### Update gst-build

This command will update all the repos and will reissue a build. Take care this command will destroy your dev
branch.
```
ninja -C build_dir update
```

#### Let's add a log line in gst-plugins-base

In this tutorial, I will explain how to add a log line in videotestsrc, gst-plugins-base's plugin, rebuild using
gst-build and test that the new log is now displayed.

##### Edit the file

```
vim subprojects/gst-plugins-base/gst/videotestsrc/gstvideotestsrc.c
```
Go to the method `gst_video_test_src_start` and edit the file by adding for example:

```
GST_ERROR_OBJECT (src, ""Starting to debug videotestsrc");
```

Then close the editor.

##### Build with gst-build

```
ninja -C build
```

You should see that only the file `gstvideotestsrc.c` was rebuilt by gst-build.

##### Test the changes

In order to enable the logs, you have to export the environment variable GST_LOG. You'll find further infos
visiting this [page](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c)

Let's start the playback and display the result in the terminal. The following command will display
all the log from videotestsrc with the category ERROR(1).

```
GST_DEBUG=videotestsrc:1 gst-launch-1.0 videotestsrc num-buffers=1 ! fakevideosink
```
You should have this output:

```
Setting pipeline to PAUSED ...
0:00:00.225273663 21743 0x565528ab7100 ERROR           videotestsrc gstvideotestsrc.c:1216:gst_video_test_src_start:<videotestsrc0> Starting to debug videotestsrc
Pipeline is PREROLLING ...
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
New clock: GstSystemClock
Got EOS from element "pipeline0".
Execution ended after 0:00:00.033464391
Setting pipeline to PAUSED ...
Setting pipeline to READY ...
Setting pipeline to NULL ...
Freeing pipeline ..
```

#### Adding a new repository

Better to be outside of `uninstalled` env.
If you want to add a new repository and work in this environment. Very simple and handy way, you'll have to:

```
# cd subprojects
# git clone my_subproject
# cd ../build
# rm -rf * && meson .. -Dcustom_subprojects=my_subproject
```

And then you can go in your subproject, edit, change, remove even stare at his beauty and then you go back to
the root of `gst-build` and issue a:

```
# ninja -C build
# ninja -C build uninstalled
```

## Conclusion

Not very sure who is best for sure and the other be dropped but I would say that cerbero is a good tool
to package or debug an existing bug where the environment is stable with the less dependencies to the system.
But for the flexibility and the rapidity, I will advise to use gst-build which can used also in a stable 
environment using a docker image by example.
