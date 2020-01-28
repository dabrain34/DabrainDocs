## My Cross Compiling Journey With gst-build and GStreamer


### The State of the Art:

GStreamer relies on multiple repositories such as base and good to build its ecosystem, and now owns more than 30 projects in Gitlab. So, a unified tool/build system has always been necessary to build a specified version.

For more than 10 years, a script named `gst-uninstalled` was present in the `gstreamer/scripts` folder to build the whole solution. Although this tool was not very flexible and was missing some options in the command line, it was good enough if you wanted to tackle a surprising bug in our favorite framework. But it was not as good at providing a real swiss-army knife approach to building  GStreamer and its dependencies.

Another build system named [cerbero](https://gitlab.freedesktop.org/gstreamer/cerbero), implemented a few years ago, provides a standalone solution to building GStreamer packages. This solution offers a wide range of options in addition to a proper sandbox to avoid system dependencies and to be able to prepare packages according to proper third party software dependencies for a given version. `cerbero` is written in Python and can build for the host machine like `gst-uninstalled` but also for various common targets depending on the host. Indeed a Linux regular desktop host will offer to cross-build GStreamer for x86(32/64bits) but also for archictecture such *ARM* and system such as *Microsoft Windows*. Despite a shell environment allowing artifact testing, `cerbero` is not really convenient for a day to day development related to GStreamer as a new plugin development or a bug fix as it is not easy to update to the last revision without loosing a current work, or to test another branch of GStreamer


### The Rise of gst-build:

In order to improve this situation,  [gst-build](https://gitlab.freedesktop.org/gstreamer/gst-build) was born .
Taking advantage of the flexibility of the rising build system, [meson](https://mesonbuild.com/), `gst-build` has been implemented to replace `gst-uninstalled` and provide a quick and smooth environment to hack into GStreamer and its dependencies.


### Autotools Is Dead, Long Live Meson:

Since GStreamer 1.18, `meson` has been chosen as the only build system for the official GStreamer repositories. For its simplicity, speed and flexibility, `meson` replaces `autotools`, so it is also perfect for use with `gst-build`. Indeed `gst-build` is first and foremost a `meson` project including `GStreamer` sub-projects with options to enable/disable selected sub-projects.

So lets take a look on how to get started with `gst-build`:


### A first step using gst-build:

#### Preliminaries

`gst-build` is mainly a `meson.build` project. It reads .wrap files which are located in the `subprojects` folder to determine the elements of the project such as GStreamer or Glib. These subprojects use the `meson` build system. `gst-build` comes with the essential projects you need to start using GStreamer and build it almost without system dependencies. BY the way, GStreamer framework does not need system dependencies as gst-build bundles `libffi` or `glib`. but can also gather the dependencies using `pkg-config` from the system to build the GStreamer plugins such as flac, for example, which needs libflac to build.

#### Environment

As we have to choose a real development environment, a 64 bit machine has been selected:

 * Ubuntu 18.04
 * Bash Shell

#### Prerequisites

 * build-essential (gcc)
 * python3
 * git
 * meson
 * ninja

#### Install meson and ninja

Here are the essential dependencies you need to install before running meson and ninja.

```
sudo apt install build-essential python3 git ninja python3-pip
```

You can now install meson from the pip repository.

```
 pip3 install --user meson
```

This will install `meson` into `~/.local/bin` which may or may not be included automatically in your PATH by default.


#### Fetch and Configure

This step will download the GStreamer repositories including some dependencies such as glib etc into the `subprojects` folder. Basically
it tries to download as many "mesonified" third party libraries as possible,  and `breaking news` the cmake ones, as a bridge
has been implemented recently if necessary.

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

This step will build all GStreamer base libraries in addition to the plugins from base/good/bad/ugly/libav if their
dependencies have been met or built by gst-build (ie glib, openh264 etc.).

```
# ninja -C build_dir
```


#### Test gst-build

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

From this environment you are now ready to use the power of GStreamer, and even implement new features in it without the fear of using out of date version.

From this shell, you are also able to compile without exiting the environment. This feature is very convenient to test a branch or fix a bug. Go to the `subprojects` folder and modify the code directly.

```
[gst-master] bash-prompt # gst-inspect-1.0
```

### Let's Talk about Cross-Compilation:

As `gst-build` is the perfect partner to get started with a GStreamer development, here is my experience to perform a cross-build, which can be very useful when you want to save precious build time or be able to work on both host and target with the same base code.

In this post, I will target an **aarch64** cpu for the Xilinx reference design: **Zynq UltraScale+ MPSoC rev F**

#### Prerequisites

* Toolchain (aarch64-linux-gnu-gcc) + sysroot (optional)
* Meson cross file
* Meson > 0.54 (which is not available on blog creation date but [patch merged](https://github.com/mesonbuild/meson/pull/6461) in the official repository)

First we'll need here to have a proper toolchain to cross-build. In my case I used the regular toolchain provided by Ubuntu installing the packages:

```
# sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu libc6-dev-arm64-cross
```

This is installing a minimal toolchain in `/usr/aarch64-linux-gnu/`

#### Cross file generated with generate-cross-file.py

Here is the cross file used to build for `aarch64`, this file has been generated with this [helper script](https://gitlab.freedesktop.org/dabrain34/gst-build/blob/dab_add_cross_file_generation/generate-cross-file.py) allowing to generated the cross file for other target as well.
As you can see, here, we don't use a dedicated rootfs because gst-build will build all that we need for the GStreamer essentials.

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

Here `meson` will use `aarch64-linux-gnu-gxx` to compile with the given args setup above. As `meson` does not recommend to use environment variables, the cross file contains hard-coded path to the sysroot to provide package config.
Indeed since `meson` > 0.54, you can define `pkg_config_libdir` which will help pkg-config to search for the package config files for the given target. You can also tell the path to the pkg-config wrapper by modifying the pkgconfig variable as well.
Predefined cross file can also be found in `gst-build/data/cross-files`


#### Configuring the project for Zynq UltraScale+ MPSoC rev F

When the cross file is ready, we can now configure `gst-build` in order to have a dedicated build for our platform. Here I am disabling some unnecessary options of gst-build such as libav, vaapi or gtk_doc

Please ensure that you have the last meson version, otherwise gst-build will take glib from the system.

```
# /path/to/meson_0_54 build-cross-arm64 --cross-file=my-meson-cross-file.txt -D omx=enabled -D sharp=disabled -D gst-omx:header_path=/opt/allegro-vcu-omx-il/omx_header -D gst-omx:target=zynqultrascaleplus -D libav=disabled -D rtsp_server=disabled -D vaapi=disabled -D disable_gst_omx=false -Dugly=disabled -Dgtk_doc=disabled

```

After this step, you should be able to build with `ninja`.

```
# ninja -C build-cross-arm64
```

#### Installing

Last but not the least, you need to install the artifacts in a given folder to be mounted by your target with NFS by example. You have to provide a **DESTDIR** variable to `ninja` and it will install in `$DESTDIR/usr/local/` as install prefix.

```
# DESTDIR=/opt/gst-build-cross-artifacts ninja install
```


#### Running the binaries on target


After mounting the folder or copying it to your target, you have to set up a few variables to be able to run GStreamer pipelines:

 * PATH=$DESTDIR/usr/local/bin:$PATH
 * LD_LIBRARY_PATH=$DESTDIR/usr/local/lib:$LD_LIBRARY_PATH
 * GST_PLUGIN_PATH=$DESTDIR/usr/local/lib/gstreamer-1.0
 * GST_OMX_CONFIG_DIR=$DESTDIR/usr/local/etc/xdg

A python script is also available [here](https://github.com/dabrain34/gstreamer-toolkit/blob/master/gst-build-helper/cross-gst-uninstalled.py) to set up the correct environment.


#### Building a dependency such as kmssink

In order to build a dependency such as `kmssink` which depends on `libdrm`. You'll need to get a proper sysroot with all the libraries which kms depends on.

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

Now you should be able to go back to the configure/build/install step and get the `kmssink` in your plugins registry.

I hope you'll enjoy the use of `gst-build`, which is for me a very powerful and adaptable tool.
A lot of options can be found on the `gst-build` [README page](https://gitlab.freedesktop.org/gstreamer/gst-build/README.md) such as the `update`
or the use of GStreamer branches.

If you would like to learn more about `gst-build` or any other parts of GStreamer, please [contact us](https://www.collabora.com/contact-us.html)
