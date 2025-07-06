

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
- In project dir: `cd Drivers && git submodule add --depth 1 https://github.com/hathach/tinyusb/`
- Code :
  - create "Core/Src/usb_descriptors.c"
  - create "Core/Inc/tusb_config.h"
  - modify "Core/Src/stm32f0xx_it.c"
    - add `tud_int_handler(0)` to `USB_IRQ_Handler()`
  - modify "Core/Src/main.c":
    - add `tud_init()` inside `main()`
    - add `tud_task()` inside while-loop in `main()`
    - add `tud_hid_get_report_cb()` and `tud_hid_get_report_cb()` callback functions 
      (for HID devices, other USB classes will have other callbacks)
  - modify "Makefile":
    - add `-DCFG_TUSB_MCU=OPT_MCU_STM32F0` to `C_DEFS`
    - add `-IDrivers/tinyusb/src` to `C_INCLUDES`
    - add `Drivers/tinyusb/*` source files to `C_SOURCES` (depends on your app)


### Replacing "Drivers" with submodules

The generated code from STM32CubeMX includes 50+ MB of "Drivers". 
Let's replace that with git submodules
([thank you @trobador!](https://mastodon.social/@trobador/114805503128160687)).

Steps: 

- Remove everything in the "Drivers" directory and add the following git submodules:
  ```sh
   rm -r Drivers/*
   git submodule add --depth 1 https://github.com/STMicroelectronics/stm32f0xx-hal-driver.git
   git submodule add --depth 1 https://github.com/STMicroelectronics/cmsis-device-f0.git
   git submodule add --depth 1 https://github.com/STMicroelectronics/cmsis-core.git


### Wiring

The Nucleo-F042K6 board does not come with a USB connector on the USB pins.
So you need to wire up a USB jack/plug yourself.

The connections are:

- 5V - wired to VIN of Nucleo board
- USB D- : USB_DM - GPIO PA11 - D10 Nucleo board 
- USB D+ : USB_DP - GPIO PA12 - D2 Nucleo board 
- Gnd - wired to Gnd

###  References:
  - https://blog.peramid.es/posts/2024-12-31-arm.html
  - https://github.com/carrotIndustries/usbkvm/tree/main/fw/usbkvm
  - https://blog.flxzt.net/posts/tinyusb-w-stm32cubeide/
