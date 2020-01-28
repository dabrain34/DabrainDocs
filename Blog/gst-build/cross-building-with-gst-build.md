## My Cross Compiling Journey With gst-build and GStreamer

### A brief Introduction

gst-build is one of the two build system used by the community to hack into the whole GStreamer solution.
A previous blogpost here has been released to present gst-build and how to hack into it.

So I will go straight to the point regarding the cross compilation with it.

Here is my experience to perform a cross-build which can be very useful when you want to save precious build time or be able to work on both host and target with the same base code.

In this post, I will target an **aarch64** CPU for the [Xilinx)(https://www.xilinx.com/) reference design: **Zynq UltraScale+ MPSoC rev F**

#### Prerequisites

* Tool-chain (aarch64-linux-gnu-gcc) + sysroot (optional)
* Meson cross file
* Meson > 0.54 (which is not available on blog creation date but [patch merged](https://github.com/mesonbuild/meson/pull/6461) in the official repository)

First we'll need here to have a proper tool-chain to cross-build. In my case I used the regular tool-chain provided by Ubuntu installing the packages:

```
# sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu libc6-dev-arm64-cross
```

This is installing a minimal tool-chain in `/usr/aarch64-linux-gnu/`

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

Here `meson` will use `aarch64-linux-gnu-gxx` to compile with the given arguments setup above. As `meson` does not recommend to use environment variables, the cross file contains hard-coded path to the sysroot to provide package config.
Indeed since `meson` > 0.54, you can define `pkg_config_libdir` which will help pkg-config to search for the package configuration files for the given target. You can also tell the path to the pkg-config wrapper by modifying the pkgconfig variable as well.
Predefined cross file can also be found in `gst-build/data/cross-files`


#### Configuring the project for Zynq UltraScale+ MPSoC rev F

When the cross file is ready, we can now configure `gst-build` in order to have a dedicated build for our platform. Here I'm disabling some unnecessary options of `gst-build` such as *libav*, *vaapi* or *gtk_doc*.

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
# DESTDIR=/opt/gst-build-cross-artifacts ninja -C build-cross-arm64 install
```


#### Running the binaries on target


After mounting the folder or copying it to your target, you have to set up a few variables to be able to run GStreamer pipelines:

 * PATH=$DESTDIR/usr/local/bin:$PATH
 * LD_LIBRARY_PATH=$DESTDIR/usr/local/lib:$LD_LIBRARY_PATH
 * GST_PLUGIN_PATH=$DESTDIR/usr/local/lib/gstreamer-1.0
 * GST_OMX_CONFIG_DIR=$DESTDIR/usr/local/etc/xdg

A python script is also available [here](https://github.com/dabrain34/gstreamer-toolkit/blob/master/gst-build-helper/cross-gst-uninstalled.py) to setup the correct environment for your environment.


#### Building a dependency such as kmssink

In order to build a dependency such as `kmssink` which depends on `libdrm`. You'll need to get a proper sysroot with all the libraries, which `kmssink` depends on.

Regarding a root file-system with `libdrm`, I generated one with [cerbero](https://gitlab.freedesktop.org/gstreamer/cerbero) where cross compiling could be described in a next blog post :)

By now the `libdrm` recipe in `cerbero` is not available but can be found in this [merge request](https://gitlab.freedesktop.org/gstreamer/cerbero/merge_requests/392)

```
# cd /opt
# git clone https://gitlab.freedesktop.org/gstreamer/cerbero
# cd cerbero
# ./cerbero-uninstalled -c config/cross-lin-arm64.cbc bootstrap
# ./cerbero-uninstalled -c config/cross-lin-arm64.cbc build libdrm

```

This should have generated a minimal root file-system in `/opt/cerbero/build/dist/linux_arm64` which can used then with `gst-build` as a base root file-system.

You can now generate a new cross file with the given root file-system as parameter.

```
# ./generate-cross-file.py --sysroot /opt/cerbero/build/dist/linux_arm64/ --no-include-sysroot
```

Here I define a *sysroot* to be be used but I'm disabling the use of `sys_root` in the cross file to avoid `meson` to tell `pkg-config` to prefix every path with this value. `cerbero` is generating pkg-config files with the sysroot path already in each pc files.

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
A lot of options can be found in the `gst-build` [README](https://gitlab.freedesktop.org/gstreamer/gst-build/README.md) such as the `update`
or the use of GStreamer branches.

If you would like to learn more about `gst-build` or any other parts of GStreamer, please [contact us](https://www.collabora.com/contact-us.html)
