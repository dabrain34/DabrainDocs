##My journey cross compiling GStreamer with gst-build


###The state of the art:

As GStreamer relies on multiple repositories, base, good etc. to build its eco-system, a unified tool/buildsystem has always been necessary to build for a given version, the whole bunch of code. GStreamer owns now more than 30 projects on gitlab.

For more than 10 years, a script named `gst-uninstalled` was present in `gstreamer/scripts` folder to build the whole solution. Not very flexible and missing some options in the command line, it was good enough if you wanted to tackle
a surprising bug in our favorite framework but not as good to provide a real swiss knife to build GStreamer and its dependencies. 

Another buildsystem named [cerbero](https://gitlab.freedesktop.org/gstreamer/cerbero) has been also implemented a few years ago, in order to provide a standalone solution to build GStreamer packages. This solution offers a wide range of options in addition to a proper sandbox to avoid system dependencies and be able to prepare package according to proper 3rd party software dependencies with given version. `Cerbero` is written in Python and able to build for the host machine as `gst-uninstalled` but also to various common targets depending on the host. Indeed a Linux regular desktop host will offer to cross-build GStreamer for x86(32/64bits) but also for archictecture such *ARM* and system such as *Microsoft Windows*.
Despite a shell environment allowing to test artifacts, `Cerbero` is not really convenient to use for a day to day development related to GStreamer as a new plugin development or a bug fix as it is not easy to update to the last revision without loosing a current work, or test another branch of GStreamer



###The raise of gst-build:

In order to tackle this breach or just improve the existing,  [gst-build](https://gitlab.freedesktop.org/gstreamer/gst-build) was born . 
Taking advantage of the flexibility of the raising buildsystem, [meson](https://mesonbuild.com/), `gst-build` has been implemented to replace `gst-uninstalled` and provide a quick and smooth environment to hack into GStreamer and its dependencies.



###Autotools is dead, long life to Meson:

Since GStreamer 1.18, `meson` has been chosen to be the only buildsystem for the official GStreamer repositories. For his simpleness, quickness and flexibility, `meson` replaces `autotools`, as it was also perfect to be used in `gst-build`.

Indeed `gst-build` is first and foremost a `meson` projet including `GStreamer` subprojects with options to enable/disable those. 

But lets take a look on how to get started with it.



### A first step using gst-build:

####A few bits about:

`gst-build` is mainly a `meson.build` project. It is reading .wrap files which are located in the `subprojects` to determine the elements of the project such as GStreamer or Glib. These subprojects are using `meson` build system. `gst-build` comes with the essential projects you need to start using GStreamer and build it almost without system dependencies. 
Speaking about it, GStreamer framework does not need system dependencies as gst-build bundles `libffi` or `glib`. but can also gather the dependencies using `pkg-config` from the system to build the GStreamer plugins such as flac by example which needs libflac to build.

####Environment

As we have to choose for a real dev env, a Linux based 64 bits machine has been selected:

 * Ubuntu 18.04
 * Bash Shell

####Prerequisites

 * build-essential (gcc)
 * python3
 * git
 * meson
 * ninja
 
####Install meson and ninja

Here is the essential dependencies, you need to install before running meson and ninja.

```
sudo apt install build-essential python3 git ninja python3-pip
```

You can now install meson from the pip repository.

```
 pip3 install --user meson
``` 

This will install `meson` into `~/.local/bin` which may or may not be included automatically in your PATH by default.


####Fetch & configure

This step will download the GStreamer repositories including some dependencies such as glib etc in the folder `subprojects`. Basically
it tries to download as much as possible 3rd parties which are using meson and `breaking news` the cmake ones, seen that a bridge has been implemented recently if necessary.

```
# git clone https://gitlab.freedesktop.org/gstreamer/gst-build
# cd gst-build
# meson build_dir
```

```
...

All GStreamer modules 1.17.0.1

  Subprojects
                        FFmpeg: YES
                         dssim: YES
                    gl-headers: YES
                      graphene: YES
                  gst-devtools: YES
          gst-editing-services: YES
                  gst-examples: YES
    gst-integration-testsuites: YES
                     gst-libav: YES
                       gst-omx: YES
               gst-plugins-bad: YES
              gst-plugins-base: YES
              gst-plugins-good: YES
                gst-plugins-rs: NO
              gst-plugins-ugly: YES
                    gst-python: NO
               gst-rtsp-server: YES
                     gstreamer: YES
               gstreamer-sharp: Feature 'sharp' disabled
               gstreamer-vaapi: YES
                         gtest: NO
                   libmicrodns: YES
                       libnice: YES
                        libpsl: YES
                       libsoup: NO
                      openh264: YES
                           orc: YES
                     pygobject: NO
                        sqlite: YES
                      tinyalsa: NO
                          x264: YES
Option buildtype is: debug [default: debugoptimized]
Found ninja-1.8.2 at /usr/bin/ninja

```

After this steps, a new folder `build_dir` should be ready to be used by `ninja` to build the binaries.

##### Build gst-build

This step will build all GStreamer base libs in addition to the plugins from base/good/bad/ugly/libav if their
depencies have been met or built by gst-build (ie glib, openh264 etc.).

```
# ninja -C build_dir
```


####Test gst-build

This command will create a new environment where all tools and plugins built previously are available in the env super-setting the system one with the right environment variables.

```
# ninja -C build_dir uninstalled
```

A prefix to  your prompt should be available as


```
[gst-master] bash-prompt #
```

```
[gst-master] bash-prompt # env | grep GST_
```

From this environment you are now ready to use the power of GStreamer even implement new features in it without the fear of using a non up2date version. 

From this shell, you are also able to compile without exiting the environment. This feature is very convenient to test a branch, a bug fix. Go to subfolder and modify the code directly in the folder.

```
[gst-master] bash-prompt # gst-inspect-1.0
```

###Let's speak about cross-compilation:

As `gst-build` is the perfect partner to get started with a GStreamer development, here is my experience to perform a cross-build which can be very useful when you want to save precious build time or be able to work on both host and target with the same base code.

In this post, I will target an **aarch64** cpu for the Xilinx reference design: **Zynq UltraScale+ MPSoC rev F**

####Prerequisites

* Toolchain (aarch64-linux-gnu-gcc) + sysroot (optional)
* Meson cross file
* Meson > 0.54 (which is not available on blog creation date but [patch merged](https://github.com/mesonbuild/meson/pull/6461) in the official repository)

First we'll need here to have a proper toolchain to cross-build. In my case I used the regular toolchain provided by Ubuntu installing the packages:

```
# sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu libc6-dev-arm64-cross
```

This is installing a minimal toolchain in `/usr/aarch64-linux-gnu/`

####Cross file generated with generate-cross-file.py

Here is the cross file used to build for `aarch64`, this file has been generated with this [helper script](https://gitlab.freedesktop.org/dabrain34/gst-build/blob/dab_add_cross_file_generation/generate-cross-file.py)
allowing to generated the cross file for other target as well.
As you can see, here, we dont use a dedicated rootfs because gst-build will build all what we need for the essential of GStreamer.

```
# ./generate-cross-file.py
```

```
[host_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'arm64'
endian = 'little'

[properties]
c_args = ['-Wall', '-g', '-O2']
cpp_args = ['-Wall', '-g', '-O2']
objc_args = ['-Wall', '-g', '-O2']
objcpp_args = []
c_link_args = []
cpp_link_args = []
objc_link_args = []
objcpp_link_args = []
pkg_config_libdir = ['__unknown_sysroot__']


[binaries]
c = ['aarch64-linux-gnu-gcc']
cpp = ['aarch64-linux-gnu-g++']
ar = ['aarch64-linux-gnu-ar']
pkgconfig = 'pkg-config'
cmake = ['false']
strip = ['aarch64-linux-gnu-strip']

```

Here `meson` will use `aarch64-linux-gnu-gxx` to compile with the given args setup above. As `meson` does not recommend to use environment variables, the cross file contains hardcoded path to the sysroot to provide package config.
Indeed since `meson` > 0.54, you can define `pkg_config_libdir` which will help pkg-config to search for the package config files for the given target. You can also tell the path to the pkg-config wrapper by modifying the pkgconfig variable as well.
Predefined cross file can also be found in `gst-build/data/cross-files`


####Configuring the project for Zynq UltraScale+ MPSoC rev F

When the cross file is ready, we can now configure `gst-build` in order to have a dedicated build for our platform. Here i'm disabling some unecessary options of gst-build such as libav, vaapi or gtk_doc 

Please ensure that you are having the last meson version otherwise gst-build will take glib from the system.

```
# /path/to/meson_0_54 build-cross-arm64 --cross-file=my-meson-cross-file.txt -D omx=enabled -D sharp=disabled -D gst-omx:header_path=/opt/allegro-vcu-omx-il/omx_header -D gst-omx:target=zynqultrascaleplus -D libav=disabled -D rtsp_server=disabled -D vaapi=disabled -D disable_gst_omx=false -Dugly=disabled -Dgtk_doc=disabled

```

After this step, you should be able to run the build with `ninja`.

```
# ninja -C build-cross-arm64
```

####Installing

Last step but not the least, you need to install the artifacts in a given folder to be mount by your target with NFS by example. You have to provide a **DESTDIR** variable to `ninja` and it will install in `$DESTDIR/usr/local/` as install prefix.

```
# DESTDIR=/opt/gst-build-cross-artifacts ninja install
```


#### Running the binaries on target


After mounting the folder or copying it to your target, you have to setup a few variables to be able to run GStreamer pipelines:

 * PATH=$DESTDIR/usr/local/bin:$PATH
 * LD_LIBRARY_PATH=$DESTDIR/usr/local/lib:$LD_LIBRARY_PATH
 * GST_PLUGIN_PATH=$DESTDIR/usr/local/lib/gstreamer-1.0
 * GST_OMX_CONFIG_DIR=$DESTDIR/usr/local/etc/xdg

A python script is also available [here](https://github.com/dabrain34/gstreamer-toolkit/blob/master/gst-build-helper/cross-gst-uninstalled.py) to setup the correct environment for your environment.
 

####Building a dependency such as kmssink

In order to build a dependency such as `kmssink` which depends on `libdrm`. You'll need to get a proper sysroot with all the libraries, kms depends on.
 
Regarding a rootfs with libdrm, I generated one with [cerbero](https://gitlab.freedesktop.org/gstreamer/cerbero) where cross compiling could be described in a next blog post :)

By now the libdrm recipe in cerbero is not available but can be found in this [merge request](https://gitlab.freedesktop.org/gstreamer/cerbero/merge_requests/392)

```
# cd /opt
# git clone https://gitlab.freedesktop.org/gstreamer/cerbero
# cd cerbero
# ./cerbero-uninstalled -c config/cross-lin-arm64.cbc bootstrap
# ./cerbero-uninstalled -c config/cross-lin-arm64.cbc build libdrm

```

This should have generated a minimal rootfs in `/opt/cerbero/build/dist/linux_arm64` which can used then with `gst-build` as a base rootfs.

You can now generate a new cross file with the given rootfs as paramater.

```
# ./generate-cross-file.py --sysroot /opt/cerbero/build/dist/linux_arm64/ --no-include-sysroot
```

Here I define a sysroot to be be used but i'm disabling the use of `sys_root` in the cross file to avoid `meson` to tell `pkg-config` to preprend every path with this value. `cerbero` is generating pkg-config files with the sysroot path already in each pc files.

```
[host_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'arm64'
endian = 'little'

[properties]
c_args = ['-Wall', '-g', '-O2']
cpp_args = ['-Wall', '-g', '-O2']
objc_args = ['-Wall', '-g', '-O2']
objcpp_args = []
c_link_args = ['-L/opt/cerbero/build/dist/linux_arm64', '-Wl,-rpath-link=/opt/cerbero/build/dist/linux_arm64']
cpp_link_args = ['-L/opt/cerbero/build/dist/linux_arm64', '-Wl,-rpath-link=/opt/cerbero/build/dist/linux_arm64']
objc_link_args = ['-L/opt/cerbero/build/dist/linux_arm64', '-Wl,-rpath-link=/opt/cerbero/build/dist/linux_arm64']
objcpp_link_args = ['-L/opt/cerbero/build/dist/linux_arm64', '-Wl,-rpath-link=/opt/cerbero/build/dist/linux_arm64']
pkg_config_libdir = ['/opt/cerbero/build/dist/linux_arm64/pkgconfig:/opt/NFS/cerbero_rootfs/linux_arm64//usr/share/pkgconfig']


[binaries]
c = ['aarch64-linux-gnu-gcc']
cpp = ['aarch64-linux-gnu-g++']
ar = ['aarch64-linux-gnu-ar']
pkgconfig = 'pkg-config'
cmake = ['false']
strip = ['aarch64-linux-gnu-strip']
```

Then you should be able to go back to the configure/build/install step and get the `kmssink` in your plugins registry.

Hope you'll enjoy the use of `gst-build`, which is for me a very powerful and adaptable tool.

A lot of options can be found on its [README page](https://gitlab.freedesktop.org/gstreamer/gst-build/README.md) such as the `update` or the use of branches in GStreamer.

