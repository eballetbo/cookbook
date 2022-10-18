# Getting Started With Raspberry Pi Pico

The [Raspberry Pi Pico board](https://www.raspberrypi.org/documentation/pico/getting-started/)
is and interesting MCU (RP2040 microcontroller) for hardware and software engineers. Recently I
got one, and as someone told me that you can use Rust with it (a language that I'm learning right
now) I couldn't wait to give it a try.

This post is just a quick guide, collecting some information a found on other places, on how to
getting started with that board and how to build programs in C/C++ and Rust for the Raspberry Pi
Pico.

Since I run [Fedora Linux](fedoraproject.org/) on my laptop, the steps on this guide works on that
distribution but should be very similar for other distributions.

## Raspberry Pi Pico C/C++ programs

First I wanted to cross compile the RP2040 example programs using the
[Raspberry Pi Pico C/C++ SDK](https://datasheets.raspberrypi.org/pico/getting-started-with-pico.pdf). But
the installation scripts assume a Raspberry Pi OS running on a Raspberry Pi 4 or an equivalent
Debian-based Linux distribution. Which is not my case. However, adapt the steps to use a
[Fedora Linux](fedoraproject.org/) distribution was pretty trivial.

### Install Fedora dependencies

My Fedora 36 system has most of the "C Development Tools and Libraries" (`dnf install @development-tools`)
packages already installed, so I might miss some package dependencies. If that's the case, please let me know.

### Get the SDK and examples

There is a [pico-examples](https://github.com/raspberrypi/pico-examples) repository that provides a set of
example applications that are written using the [pico-sdk](https://github.com/raspberrypi/pico-sdk). Following (and maybe
adapting a bit) these instructions you should be able to setup the environment. In my case I followed these steps:

First, create a working directory

```sh
mkdir ~/pico
cd ~/pico
```

Then, clone the [pico-sdk](https://github.com/raspberrypi/pico-sdk) and [pico-examples](https://github.com/raspberrypi/pico-examples) git repositories.

```sh
git clone https://github.com/raspberrypi/pico-sdk.git
cd pico-sdk
git submodule update --init
cd ..
git clone https://github.com/raspberrypi/pico-examples.git
```

### Install the ARM cross compiler toolchain

As the MCU of the Raspberry Pi Pico is based on a RP2040 microcontroller, a different architecture that the one is running in my laptop, we
need to install a cross toolchain to build the binaries. Here's a minimal list of dependencies that will pull in additional required packages.

```
dnf install gcc-arm-linux-gnu \
 arm-none-eabi-gcc-cs-c++ \
 arm-none-eabi-gcc-cs \
 arm-none-eabi-binutils \
 arm-none-eabi-newlib
```

### Blinking an LED in C

From the pico directory we created earlier, cd into pico-examples and create a build directory

```sh
cd pico-examples
mkdir build
cd build
```

Then, assuming you cloned the pico-sdk and pico-examples repositories into the same directory side-by-side, set the
PICO_SDK_PATH:

```sh
export PICO_SDK_PATH=../../pico-sdk
```

Prepare your cmake build directory by running
```sh
cmake ..
```

CMake has now prepared a build area for the pico-examples tree. From here, it is possible to type make to build all example
applications. However, as we are building blink we will only build that application for now by changing directory into the
blink directory before typing make.

```sh
cd blink
make -j4
```

Amongst other targets, we have now built:

* blink.elf, which is used by the debugger
* blink.uf2, which can be dragged onto the RP2040 USB Mass Storage Device

This binary will blink the on-board LED of the Raspberry Pi Pico which is connected to GPIO25 of RP2040.

### Flashing the "blink.uf2" Binary

The simplest method to load software onto a RP2040-based board is by mounting it as a USB Mass Storage Device. Connect your Fedora laptop to your Raspberry Pi Pico using a Micro-USB cable, making sure that you hold down the BOOTSEL button to force it into USB Mass Storage mode.

```sh
$ dmesg | tail
[ 371.973555] sd 0:0:0:0: [sda] Attached SCSI removable disk
```

Check that the USB mass storage was detected and mounted
```sh
$ ls /run/media/$username/RPI-RP2
INDEX.HTM INFO_UF2.TXT
```

 You can copy the **blink.uf2** file onto the board to program the flash.
```sh
cp blink.uf2 /mnt/pico
```

 The RP2040 will reboot, unmounting itself as a Mass Storage Device, and start to run the flashed code.

## Raspberry Pi Pico Rust programs

The microprocessor typically uses C/C++, or MicroPython. This doesn’t mean, however, that we cannot get Rust to run on the Raspberry Pi Pico.

As my Fedora rust installation has no support fo the `thumbv6m-none-eabi` target I decided to use `rustup` to have the latest tools.

To do so, run the following commands:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

And then, add the `thumbv6m-none-eabi`target with:

```sh
rustup target add thumbv6m-none-eabi
```

Rust can build to elf executable files. The problem is, elf executables will not run when dropped into the Pico. To run code, we need to translate
it to the `uf2` format. This is the format that usually works when dragging and dropping files into the Raspberry Pi Pico.

Fortunetely, `elf2uf2-rs` accomplishes this for us. This will automate the deployment of our code. This way, we wont have to deal with
flashing the code to the device, we can just have cargo build our project and the code will be deployed on the Raspberry Pi Pico automatically.

So, install the `elf2uf2-rs` crate package with:

```sh
cargo install elf2uf2-rs
```

And add the following to your cargo config (mine is located at ~/.cargo/config

```txt
[target.thumbv6m-none-eabi]
runner = "elf2uf2-rs -d"
```

To keep things simple, I’ll be using example code from the repository that will blink an LED on our board. Specifically I will use the blinking
onboard LED example.

Clone the `rp-rs/rp-hal` crate and navigate into the examples under the `rp-hal/boards/rp-pico` directory:

```sh
git clone https://github.com/rp-rs/rp-hal.git
cd rp-hal
```

Now, making sure that you hold down the BOOTSEL button to force it into USB Mass Storage mode, we can
load the example onto our Raspberry Pi Pico using the following shell command:

```sh
cargo run --release --example pico_blinky
```

## Conclusion

The Raspberry Pi Pico looks very promising, while undercuts most on the market while delivering unique features. Programming and getting started with
it is pretty simple, and things just work. Not to mention the exhaustive documentation that goes along with it. I'm pretty sure I'm going to have fun
with this microcontroller and probly I'm going to use as a platform to learn more about Rust.
