# Используем для отладки на маке

* c-periphery-al(там правки для мака)  используется вместо c-periphery
* git clone --recurse-submodules  https://github.com/alexandrlevashov/lua-periphery.git
* git submodule update --init --recursive
* make clean all
* далее положить рядом с lua скриптом или в папку библиотек periphery.so


# lua-periphery [![Build Status](https://github.com/vsergeev/lua-periphery/actions/workflows/build.yml/badge.svg)](https://github.com/vsergeev/lua-periphery/actions/workflows/build.yml) [![GitHub release](https://img.shields.io/github/release/vsergeev/lua-periphery.svg?maxAge=7200)](https://github.com/vsergeev/lua-periphery) [![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/vsergeev/lua-periphery/blob/master/LICENSE)

## Linux Peripheral I/O (GPIO, LED, PWM, SPI, I2C, MMIO, Serial) with Lua

lua-periphery is a library for GPIO, LED, PWM, SPI, I2C, MMIO, and Serial peripheral I/O interface access in userspace Linux. It is useful in embedded Linux environments (including Raspberry Pi, BeagleBone, etc. platforms) for interfacing with external peripherals. lua-periphery is compatible with Lua 5.1 (including LuaJIT), Lua 5.2, Lua 5.3, and Lua 5.4, has no dependencies outside the standard C library and Linux, is portable across architectures, and is MIT licensed.

Using Python or C? Check out the [python-periphery](https://github.com/vsergeev/python-periphery) and [c-periphery](https://github.com/vsergeev/c-periphery) projects.

Contributed libraries: [java-periphery](https://github.com/sgjava/java-periphery), [dart_periphery](https://github.com/pezi/dart_periphery)

## Examples

### GPIO

``` lua
local GPIO = require('periphery').GPIO

-- Open GPIO /dev/gpiochip0 line 10 with input direction
local gpio_in = GPIO("/dev/gpiochip0", 10, "in")
-- Open GPIO /dev/gpiochip0 line 12 with output direction
local gpio_out = GPIO("/dev/gpiochip0", 12, "out")

local value = gpio_in:read()
gpio_out:write(not value)

gpio_in:close()
gpio_out:close()
```

[Go to GPIO documentation.](docs/gpio.md)

### LED

``` lua
local LED = require('periphery').LED

-- Open LED led0
local led = LED("led0")

-- Turn on LED (set max brightness)
led:write(true)

-- Set half brightness
led:write(math.floor(led.max_brightness / 2))

-- Turn off LED (set zero brightness)
led:write(false)

led:close()
```

[Go to LED documentation.](docs/led.md)

### PWM

``` lua
local PWM = require('periphery').PWM

-- Open PWM chip 0, channel 10
local pwm = PWM(0, 10)

-- Set frequency to 1 kHz
pwm.frequency = 1e3
-- Set duty cycle to 75%
pwm.duty_cycle = 0.75

-- Enable PWM output
pwm:enable()

pwm:close()
```

[Go to PWM documentation.](docs/pwm.md)

### SPI

``` lua
local SPI = require('periphery').SPI

-- Open spidev1.0 with mode 0 and max speed 1MHz
local spi = SPI("/dev/spidev1.0", 0, 1e6)

local data_out = {0xaa, 0xbb, 0xcc, 0xdd}
local data_in = spi:transfer(data_out)

print(string.format("shifted out {0x%02x, 0x%02x, 0x%02x, 0x%02x}", unpack(data_out)))
print(string.format("shifted in  {0x%02x, 0x%02x, 0x%02x, 0x%02x}", unpack(data_in)))

spi:close()
```

[Go to SPI documentation.](docs/spi.md)

### I2C

``` lua
local I2C = require('periphery').I2C

-- Open i2c-0 controller
local i2c = I2C("/dev/i2c-0")

-- Read byte at address 0x100 of EEPROM at 0x50
local msgs = { {0x01, 0x00}, {0x00, flags = I2C.I2C_M_RD} }
i2c:transfer(0x50, msgs)
print(string.format("0x100: 0x%02x", msgs[2][1]))

i2c:close()
```

[Go to I2C documentation.](docs/i2c.md)

### MMIO

``` lua
local MMIO = require('periphery').MMIO

-- Open am335x real-time clock subsystem page
local rtc_mmio = MMIO(0x44E3E000, 0x1000)

-- Read current time
local rtc_secs = rtc_mmio:read32(0x00)
local rtc_mins = rtc_mmio:read32(0x04)
local rtc_hrs = rtc_mmio:read32(0x08)

print(string.format("hours: %02x minutes: %02x seconds: %02x", rtc_hrs, rtc_mins, rtc_secs))

rtc_mmio:close()

-- Open am335x control module page
local ctrl_mmio = MMIO(0x44E10000, 0x1000)

-- Read MAC address
local mac_id0_lo = ctrl_mmio:read32(0x630)
local mac_id0_hi = ctrl_mmio:read32(0x634)

print(string.format("MAC address: %04x%08x", mac_id0_lo, mac_id0_hi))

ctrl_mmio:close()
```

[Go to MMIO documentation.](docs/mmio.md)

### Serial

``` lua
local Serial = require('periphery').Serial

-- Open /dev/ttyUSB0 with baudrate 115200, and defaults of 8N1, no flow control
local serial = Serial("/dev/ttyUSB0", 115200)

serial:write("Hello World!")

-- Read up to 128 bytes with 500ms timeout
local buf = serial:read(128, 500)
print(string.format("read %d bytes: _%s_", #buf, buf))

serial:close()
```

[Go to Serial documentation.](docs/serial.md)

### Error Handling

lua-periphery errors are descriptive table objects with an error code string, C errno, and a user message.

``` lua
-- Example of error caught with pcall()
> status, err = pcall(function () spi = periphery.SPI("/dev/spidev1.0", 0, 1e6) end)
> =status
false
> dump(err)
{
  message = "Opening SPI device \"/dev/spidev1.0\": Permission denied [errno 13]",
  c_errno = 13,
  code = "SPI_ERROR_OPEN"
}
> 

-- Example of error propagated to user
> periphery = require('periphery')
> spi = periphery.SPI('/dev/spidev1.0', 0, 1e6)
Opening SPI device "/dev/spidev1.0": Permission denied [errno 13]
> 
```

#### Note about Lua 5.1

Lua 5.1 does not automatically render the string representation of error
objects that are reported to the console, and instead shows the following error
message:

``` lua
> periphery = require('periphery')
> gpio = periphery.GPIO(14, 'in')
(error object is not a string)
> 
```

These errors can be caught with `pcall()` and rendered in their string
representation with `tostring()`, `print()`, or by evaluation in the
interactive console:

``` lua
> periphery = require('periphery')
> gpio, err = pcall(periphery.GPIO, 14, 'in')
> =tostring(err)
Opening GPIO: opening 'export': Permission denied [errno 13]
> print(err)
Opening GPIO: opening 'export': Permission denied [errno 13]
> =err
Opening GPIO: opening 'export': Permission denied [errno 13]
> 
```

This only applies to Lua 5.1. LuaJIT and Lua 5.2 onwards automatically render
the string representation of error objects that are reported to the console.

## Documentation

`man` page style documentation for each interface is available in [docs](docs/) folder.

## Installation

#### Build and install with LuaRocks

``` console
$ sudo luarocks install lua-periphery
```

#### Build and install from source

Clone lua-periphery recursively to also fetch [c-periphery](https://github.com/vsergeev/c-periphery), which lua-periphery is built on.

``` console
$ git clone --recursive https://github.com/vsergeev/lua-periphery.git
$ cd lua-periphery
$ make clean all
$ sudo make install
```

lua-periphery can then be loaded in lua with `periphery = require('periphery')`.

#### Cross-compiling from Source

Set the `CROSS_COMPILE` environment variable with the cross-compiler prefix when calling make. Your target's sysroot must provide the Lua includes.

``` console
$ CROSS=arm-linux- make clean all
cd c-periphery && make clean
make[1]: Entering directory '/home/anteater/projects/software/lua-periphery/c-periphery'
rm -rf periphery.a obj tests/test_serial tests/test_i2c tests/test_gpio_sysfs tests/test_mmio tests/test_spi tests/test_gpio
make[1]: Leaving directory '/home/anteater/projects/software/lua-periphery/c-periphery'
rm -rf periphery.so
cd c-periphery; make
make[1]: Entering directory '/home/anteater/projects/software/lua-periphery/c-periphery'
mkdir obj
arm-linux-gcc -std=gnu99 -pedantic -Wall -Wextra -Wno-unused-parameter  -fPIC -DPERIPHERY_VERSION_COMMIT=\"v2.0.1\"  -c src/gpio.c -o obj/gpio.o
arm-linux-gcc -std=gnu99 -pedantic -Wall -Wextra -Wno-unused-parameter  -fPIC -DPERIPHERY_VERSION_COMMIT=\"v2.0.1\"  -c src/spi.c -o obj/spi.o
arm-linux-gcc -std=gnu99 -pedantic -Wall -Wextra -Wno-unused-parameter  -fPIC -DPERIPHERY_VERSION_COMMIT=\"v2.0.1\"  -c src/i2c.c -o obj/i2c.o
arm-linux-gcc -std=gnu99 -pedantic -Wall -Wextra -Wno-unused-parameter  -fPIC -DPERIPHERY_VERSION_COMMIT=\"v2.0.1\"  -c src/mmio.c -o obj/mmio.o
arm-linux-gcc -std=gnu99 -pedantic -Wall -Wextra -Wno-unused-parameter  -fPIC -DPERIPHERY_VERSION_COMMIT=\"v2.0.1\"  -c src/serial.c -o obj/serial.o
arm-linux-gcc -std=gnu99 -pedantic -Wall -Wextra -Wno-unused-parameter  -fPIC -DPERIPHERY_VERSION_COMMIT=\"v2.0.1\"  -c src/version.c -o obj/version.o
arm-linux-ar rcs periphery.a obj/gpio.o obj/spi.o obj/i2c.o obj/mmio.o obj/serial.o obj/version.o
make[1]: Leaving directory '/home/anteater/projects/software/lua-periphery/c-periphery'
arm-linux-gcc  -std=c99 -pedantic -D_XOPEN_SOURCE=700 -Wall -Wextra -Wno-unused-parameter  -fPIC -I.  -Iinc  -shared src/lua_periphery.c src/lua_mmio.c src/lua_
gpio.c src/lua_spi.c src/lua_i2c.c src/lua_serial.c c-periphery/periphery.a -o periphery.so
$ file periphery.so
periphery.so: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, not stripped
$
```

## Testing

The tests located in the [tests](tests/) folder may be run under Lua to test the correctness and functionality of lua-periphery. Some tests require interactive probing (e.g. with an oscilloscope), the installation of a physical loopback, or the existence of a particular device on a bus. See the usage of each test for more details on the required setup.

## License

lua-periphery is MIT licensed. See the included [LICENSE](LICENSE) file.

