# Picocalc_SD_Boot 


Modified for ILI9488

```
[here]
    mkdir -p ~/pico
    cd ~/pico
    git clone https://github.com/raspberrypi/pico-sdk.git
    cd pico-sdk
    git submodule update --init

export PICO_SDK_PATH="~/pico/pico-sdk"

cd ../  
    
git clone https://github.com/markbirss/Picocalc_SD_Boot.git
cd Picocalc_SD_Boot
git submodule update --init --recursive

cd ./src

mkdir build; cd build
cmake \
    -DPICO_BOARD=pico2 \
    -DPICO_PLATFORM=rp2350-arm-s \
    ..  
make    
sudo cp picocalc_sd_boot_pico2_w.uf2 /media/user/RP2350/
 
[start compile again]
cd ..
rm -fr build

```

```
Connection
// GPIOs for SPI interface (SD card)
#define SD_SPI0         0
#define SD_SCLK_PIN     18
#define SD_MOSI_PIN     19
#define SD_MISO_PIN     16
#define SD_CS_PIN       17
#define SD_DET_PIN      -1 //22

#define LCD_SPI1    1
#define LCD_SCK_PIN 10
#define LCD_MOSI_PIN 11
#define LCD_MISO_PIN 12
#define LCD_CS_PIN 13
#define LCD_DC_PIN 14
#define LCD_RST_PIN 15

// GPIOs for audio output
#define AUDIO_LEFT     28
#define AUDIO_RIGHT    27

// GPIOs for buttons
#define NEXT_BUTTON    2
#define PART_BUTTON    3

// Pico-internal GPIOs
#define PICO_PS        23
#define LED_PIN        25
```

`Picocalc_SD_Boot` is a custom bootloader for the Raspberry Pi Pico. This bootloader provides the functionality to load and execute applications from an SD card, designed to enable PicoCalc to load firmware to the Pico using an SD card easily.

<div align="center">
    <img src="img/sd_boot.jpg" alt="sdboot" width="80%">
</div>

## 🚧 Improvement Plans
work in progress plans [Feature Request Post](https://forum.clockworkpi.com/t/i-made-an-app-that-dynamically-load-firmware-from-sd-card/16664/25?u=adwuard)
- [ ] Avoiding the need to recompile Apps, A/B image see [Address Translation (see page 364 of the RP2350 datasheet, section 5.1.19)](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf)
- [ ] Support for .uf2 files, extract `.bin` from `.uf2` firmware  [pico-bootrom](https://github.com/raspberrypi/pico-bootrom-rp2350)
- [ ] USB Mass Storage mode for SD card. [Related Demo Code](https://github.com/hathach/tinyusb/tree/master/examples/device/cdc_msc/src)



## Prebuilt .uf2 Firmware Available
[![Download Prebuilt Firmware](https://img.shields.io/badge/Download-Firmware-blue)](https://github.com/adwuard/Picocalc_SD_Boot/tree/main/prebuild_output)

Click the button above to download the prebuilt `.uf2` firmware for the `Picocalc_SD_Boot ` bootloader. Flash your pico with holding BOOT_SEL and drag and drop the `.uf2` file.


## Bootloader Build From Scratch
Clone the source code and initialize the submodules.

```bash
git clone https://github.com/adwuard/Picocalc_SD_Boot.git
cd Picocalc_SD_Boot
git submodule update --init --recursive
```

Build the bootloader.

```bash
cd ./src
mkdir build; cd build
cmake ..
make
```

## SD Card Application Build and Deployment
🚨 **Important Note:** 🚨  
```
Applications intended for SD card boot "MUST REBUILD" using a custom linker script to accommodate the program's offset address. 
```
✅ You can find all the recompiled firmware available in the [sd_card_content](https://github.com/adwuard/Picocalc_SD_Boot/tree/main/sd_card_content) folder.

--- 
This section explains how to build and deploy applications on an SD card. Below is a simple example of a CMakeLists.txt for an application.


## Step 1 Copy Custom Link Script
Copy `memmap_sdcard_app.ld` to your project repository.

## Step 2 Add Custom Link Script to CMakeList.txt
```CMakeLists.txt
cmake_minimum_required(VERSION 3.13...3.27)
include(vendor/pico_sdk_import.cmake)
add_subdirectory(pico-sdcard-boot)

project(hello)
set(FAMILY rp2040)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
pico_sdk_init()

add_executable(hello main.c)
target_link_libraries(hello PRIVATE pico_stdlib)
pico_enable_stdio_usb(hello 1)
pico_add_extra_outputs(hello)


# ----------- COPY THIS Section -----------
function(enable_sdcard_app target)
  #pico_set_linker_script(${target} ${CMAKE_SOURCE_DIR}/memmap_sdcard_app.ld)
  if(${PICO_PLATFORM} STREQUAL "rp2040")
    pico_set_linker_script(${CMAKE_PROJECT_NAME} ${CMAKE_SOURCE_DIR}/memmap_default_rp2040.ld)
  elseif(${PICO_PLATFORM} MATCHES "rp2350")
    pico_set_linker_script(${CMAKE_PROJECT_NAME} ${CMAKE_SOURCE_DIR}/memmap_default_rp2350.ld)
  endif()
endfunction()
# ----------- COPY THIS Section END -----------
```
The `enable_sdcard_app()` function sets the necessary `memmap_sdcard_app.ld` linker script for projects that boot from an SD card.

### Build and Deployment Process
1. Build the project using the above CMakeLists.txt.

```bash
mkdir build; cd build
PICO_SDK_PATH=/path/to/pico-sdk cmake ..
make
```

## Step 3 Your Custom Application Is Ready For SD Card Boot 
Once the build is complete, copy the generated `APP_FW.bin` file to the `/sd` directory of the SD card.



## Technical Implementation Notes
### Flash Update Mechanism
The bootloader implements a safe update mechanism with the following features:

- The bootloader itself resides in a protected flash area (first 256KB) that is never overwritten during updates
- Only the application region of flash (starting at 256KB) is updated using `flash_range_erase` and `flash_range_program`
- The bootloader verifies if the update differs from the current flash image before performing any write operations
- Flash programming operations are executed from RAM using the `__not_in_flash_func` attribute to ensure safe execution while the flash is being modified
- Program size is verified to prevent overwriting critical memory regions


### Flash Programming Safety
When updating flash memory, the code that performs the flash operations must not be executed from the flash itself. The bootloader ensures this by:

1. Using the Raspberry Pi Pico SDK's `__not_in_flash_func` attribute for all flash programming functions
2. This attribute ensures the functions are executed from RAM rather than flash
3. Without this protection, attempting to modify the flash while executing code from it could lead to unpredictable behavior or bricking the device

## Credits
- [Hiroyuki Oyama](https://github.com/oyama/pico-sdcard-boot): Special thanks for the firmware loader mechanism and VFS file system.
  - https://github.com/oyama/pico-sdcard-boot
  - https://github.com/oyama/pico-vfs
- [TheKiwil](https://github.com/TheKiwil/): Special thanks for contributions on supporting pico2 boards with new custom linker script.

## Read More
- Blog on this repository, and more technical detail about bootloader. -->[Blog Page](https://hsuanhanlai.com/writting-custom-bootloader-for-RPI-Pico/)
- Fourm Page and Discussion: [Clockwork Pi Fourm](https://forum.clockworkpi.com/t/i-made-an-app-that-dynamically-load-firmware-from-sd-card/16664/24)
