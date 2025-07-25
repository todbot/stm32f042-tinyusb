

## stm32f042_tinyusb: TinyUSB with STM32F042 Nucleo-F042K6 board

Simple example of taking a STM32F042 project configured with STM32CubeMX
code generator and adding [TinyUSB](https://github.com/hathach/tinyusb/) 
to it to make a HID USB device.


### Creation of and Modification to Generated Code

Let's use STM32CubeMX to configure the chip, enable USB, and generate
a basic project.  Then we'll modify it.

Steps:

- In CubeMX:
  - Enable "USB" and "Device (FS)" in Connectivity->USB
  - Enable "USB global interrupt" in Connectivity->USB->NVIC
  - Generate code
- In generated code project dir:
  - add tinyusb: `cd Drivers && git submodule add --depth 1 https://github.com/hathach/tinyusb/`
  - convert from DOS format: `dos2unix * Core/**/*`
- Code changes:
  - create "Core/Src/usb_descriptors.c"  *(copy from a tinyusb hid_generic_inout example & tweak)*
  - create "Core/Inc/tusb_config.h"  *(copy from a tinyusb hid_generic_inout example & tweak)*
  - modify "Core/Src/stm32f0xx_it.c"
    - add `tud_int_handler(0)` to `USB_IRQ_Handler()`
  - modify "Core/Src/main.c":
    - add `tud_init()` inside `main()`
    - add `tud_task()` inside while(1)-loop in `main()`
    - add `tud_hid_get_report_cb()` and `tud_hid_get_report_cb()` callback functions 
      (for HID devices, other USB classes will have other callbacks)
  - modify "Makefile":
    - add `usb_descriptors.c` to `C_SOURCES`
    - add `Core/Src/syscalls.c` to `C_SOURCES` since STM forgot
    - add `Drivers/tinyusb/*` source files to `C_SOURCES` (depends on your app)
    - add `-IDrivers/tinyusb/src` to `C_INCLUDES`
    - add `-DCFG_TUSB_MCU=OPT_MCU_STM32F0` to `C_DEFS`

This repo is the result of those changes.


### Replacing "Drivers" with submodules

The generated code from STM32CubeMX includes 50+ MB of "Drivers". 
Let's replace that with git submodules
([thank you @trobador!](https://mastodon.social/@trobador/114805503128160687)).

Steps: 

- Remove everything in the "Drivers" directory and add the following git submodules:
  ```sh
   rm -rf Drivers 
   mkdir Drivers && cd Drivers
   git submodule add --depth 1 https://github.com/STMicroelectronics/stm32f0xx-hal-driver.git
   git submodule add --depth 1 https://github.com/STMicroelectronics/cmsis-device-f0.git
   git submodule add --depth 1 https://github.com/STMicroelectronics/cmsis-core.git
   git submodule add --depth 1 https://github.com/hathach/tinyusb/
   ```

- Modify Makefile to point to slightly new locations for the CMSIS and HAL files.
  Mostly this is search & replace in `C_SOURCES` and `INCLUDES` of: 
  - `Drivers/STM32F0xx_HAL_Driver` to `Drivers/stm32f0xx-hal-driver`
  - `Drivers/CMSIS/Include` to `Drivers/cmsis-device-f0` and `Drivers/cmsis-core`

This repo contains these changes. 

### Wiring

The Nucleo-F042K6 board does not come with a USB connector on the USB pins.
So you need to wire up a USB jack/plug yourself.

The connections are:

- 5V - wired to VIN of Nucleo board
- USB D- : USB_DM - GPIO PA11 - D10 Nucleo board 
- USB D+ : USB_DP - GPIO PA12 - D2 Nucleo board 
- Gnd - wired to Gnd


### Compiling on checkout

If you just want to try this code out, check out this repo, init the submodules, 
and compile:

```sh
git clone https://github.com/todbot/stm32f042_tinyusb
cd stm32f042_tinyusb
git submodule update --init
make -j4 
make flash
```

The Makefile assumes you have the following installed

- [arm-none-eabi-gcc](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
- [openocd](https://openocd.org/)
- [dfu-util](https://dfu-util.sourceforge.net/) (optional)


### Testing

I like using [`hidapitester`](https://github.com/todbot/hidapitester) to test
USB HID devices:

- Send a 40-byte Output report: 
```sh
hidapitester --vidpid CAFE:1234 -l 40 --open --send-output 0x76,0x10,0x20,0x30 
```

- Send a 40-byte Feature report on reportId 0x76:
```sh
 hidapitester --vidpid CAFE:1234 -l 41 --open --send-feature 0x76,0x10,0x20,0x30,0x40,0x50,0x55,0xAA 
```

- Receive a 40-byte Feature report on reportId 0x75:
```
hidapitester --vidpid CAFE:1234 -l 40 --open --read-feature 0x75
```

###  References:
  - https://blog.peramid.es/posts/2024-12-31-arm.html
  - https://github.com/carrotIndustries/usbkvm/tree/main/fw/usbkvm
  - https://blog.flxzt.net/posts/tinyusb-w-stm32cubeide/
