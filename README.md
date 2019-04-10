Using GCC and Makefiles on macOS to build STM32CubeMX projects
================================================================

As of v4.21.0, [STM32CubeMX] is now capable of generating Makefiles that can be used to build projects using the [GNU ARM Embedded Toolchain]. Makefiles allow you to be IDE independent and use you favorite text editor. For some people, IDEs are slow and take up a lot of resources. With a Makefile, building your project is as simple as typing `make` in your Terminal be you in Linux, Mac, or Windows. No more restrictions.

Although this tutorial has been written with macOS in mind, similar steps can be applied to Linux or Windows machines.

Detail
------

- macOS : High Sierra (10.13.6)
- VSCode : 1.33.0
- VSCode - Cortex Debug plugin : 0.2.3
- [STM32CubeMX] : 5.1.0
- [STM32CubeProgrammer] : 2.0.0
- A hardware development board: STM32F103RC board
- ST-Link V2 dongle with SWO modification (see [lujji](https://lujji.github.io/blog/stlink-clone-trace/)'s post)
- macOS Command Line Tools (CLT)
- [Homebrew] package manager (recommended to install gcc-arm-embedded, openOCD and stlink)
- [GNU ARM Embedded Toolchain] : 8-2018-q4-major
- [OpenOCD] : 0.10.0
- [texane/stlink] : 1.5.1


0 - Installing the toolchain
----------------------------
### Requirements:
- [STM32CubeMX] to generate project templates
- [STM32CubeProgrammer] to easily program STM32 products using a GUI
- A hardware development board (*e.g.* a [NUCLEO-L476RG] board)
- macOS Command Line Tools (CLT)
- [Homebrew] package manager (recommended to install gcc-arm-embedded, openOCD and stlink)
- [GNU ARM Embedded Toolchain] (arm-none-eabi) for compiler and other tools
- [OpenOCD] (>= 0.10.0) or [texane/stlink] for programming and running a GDB server


1. Install Xcode Command Line Tools (CLT). This will install *Make* and other UNIX goodies:
```
$ xcode-select --install
```
After the *Command Line Tools* were successfully installed, the remaining toolchain requirements can be installed using *Homebrew*.

2. Install *Homebrew*. Follow instructions available on [brew.sh][Homebrew]
3. Install GCC ARM Embedded Toolchain:
```
$ brew install caskroom/cask/gcc-arm-embedded
$ arm-none-eabi-gcc --version
arm-none-eabi-gcc (GNU Tools for Arm Embedded Processors 7-2017-q4-major) 7.2.1 20170904 (release) [ARM/embedded-7-branch revision 255204]
Copyright (C) 2017 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

4. Install OpenOCD:
```
$ brew install openocd
$ openOCD --version
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
    http://openocd.org/doc/doxygen/bugs.html
```

5. Install open source [texane/stlink]:
```
$ brew install stlink
$ st-info --version
v1.4.0
```

6. Install [STM32CubeMX]. After Downloading the installer, extract the archieve and try to run the macOS installer. If the macOS installer doesn't work, use the following command to manually launch the install. The procedure is described in the **STM32CubeMX User Manual** [UM1718].
```
$ cd ~/Downloads/
$ unzip en.stm32cubemx.zip -d en.stm32cubemx
$ cd en.stm32cubemx
$ java -jar SetupSTM32CubeMX-4.26.1.exe
```

7. Install [STM32CubeProgrammer]. Similarly to CubeMX, if the installer doesn't work, use:

```
$ unzip en.stm32cubeprog.zip -d en.stm32cubeprog
$ cd en.stm32cubeprog
$ java -jar SetupSTM32CubeProgrammer-1.1.0.exe
```
If the above command does not work, you could try installing `java8`:
```
$ brew tap caskroom/versions
$ brew cask install java8
```
And Java 8 will be installed at `/Library/Java/JavaVirtualMachines/jdk1.8.xxx.jdk/`. You can then use the full java path to use version 1.8 to launch the  STM32CubeProgrammer setup.
```
$ /Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/bin/java -jar SetupSTM32CubeProgrammer-1.0.0.exe
```
**Pro Tip**: Create a symbolic link to one of the binary directory searched by your `$PATH` variable:
```
$ ln -sv /Applications/STMicroelectronics/STM32Cube/STM32CubeProgrammer/STM32CubeProgrammer.app/Contents/MacOs/bin/STM32_Programmer_CLI /usr/local/bin/
```
Then, `STM32_Programmer_CLI` can be invoked diectly without having to specify the full path:
```
$ STM32_Programmer_CLI
      -------------------------------------------------------------------
                        STM32CubeProgrammer v1.1.0
      -------------------------------------------------------------------


Usage : 
STM32_Programmer_CLI.exe [command_1] [Arguments_1][[command_2] [Arguments_2]...]
```

1 - Create a Project using CubeMX
---------------------------------
If you are also using an NUCLEO-L476RG, you can use the example *"blinky"* project by cloning the following repo:
```
$ git clone https://github.com/glegrain/STM32-with-macOS.git
$ cd STM32-with-macOS/Example_Project
```


Alternatively, you can generate you own project:
1. Create a **New Project**, and select your part number or development board
2. Configure your **Pins**, **Clock Settings** and **Peripherals**
3. When you click **Project->Generate Code**, the **Project Settings** window will show up. Under **Toolchain / IDE**, select **Makefile**.

![Project Settings](images/Project_Settings.png)

For more information, refer to the **STM32CubeMX User Manual** available on [st.com](http://www.st.com/resource/en/user_manual/dm00104712.pdf). Usefull sections include:
- Tutorial 1: From pinout to project C code generation using an STM32F4 MCU
- Tutorial 4: Example of UART communications with a STM32L053xx Nucleo board

2 - Configure your Makefile
-----------------------------
Unfortunately, Makefiles generated by CubeMX do not work *out-of-the-box*. You need to edit the file and set your compiler path. Luckily, this step only has to be done once. Later on, if you want to add source, header files or simply change your compiler options, refer to the [Editing your Makefile] section for more details.

1. Locate the **ARM Embedded GCC** compiler binary location:
```
$ which arm-none-eabi-gcc
/usr/local/bin/arm-none-eabi-gcc
```

2. Open up the **Makefile** with your favorite text editor to set the `BINPATH` variable to the location of your compiler returned above:
```Makefile
#######################################
# binaries
#######################################
BINPATH = /usr/local/bin
PREFIX = arm-none-eabi-
CC = $(BINPATH)/$(PREFIX)gcc
AS = $(BINPATH)/$(PREFIX)gcc -x assembler-with-cpp
CP = $(BINPATH)/$(PREFIX)objcopy
AR = $(BINPATH)/$(PREFIX)ar
SZ = $(BINPATH)/$(PREFIX)size
HEX = $(CP) -O ihex
BIN = $(CP) -O binary -S
```
**Pro Tip**: To make the Makefile more portable between different users and environment, you can remove the `BINPATH` variable and edit the `CC`, `AS`, `CP`, `AR`, `SZ` as shown bellow. This way, *make* will look for binaries in your environment (*i.e.* executables located in your `$PATH` setting):
```Makefile
#######################################
# binaries
#######################################
PREFIX = arm-none-eabi-
CC = $(PREFIX)gcc
AS = $(PREFIX)gcc -x assembler-with-cpp
CP = $(PREFIX)objcopy
AR = $(PREFIX)ar
SZ = $(PREFIX)size
HEX = $(CP) -O ihex
BIN = $(CP) -O binary -S
```

3 - Building your project
-------------------------
In a Terminal, navigate to your project's root directory (or Makefile location). Then use the `make` command to invoke the Makefile to compile your project:
```
$ cd ~/path/to/Example_Project
$ make
```
**Pro Tip**: for faster build time, `make` can be invoked using [parallel build](https://www.gnu.org/software/make/manual/make.html#Parallel) with the `-j` option:
```
$ make -j 4
```

If all goes well, you should see a result without any errors or warning:
```
$ make
...
arm-none-eabi-size build/Example_Project.elf
   text    data     bss     dec     hex filename
   8880      24    1688   10592    2960 build/Example_Project.elf
arm-none-eabi-objcopy -O ihex build/Example_Project.elf build/Example_Project.hex
arm-none-eabi-objcopy -O binary -S build/Example_Project.elf build/Example_Project.bin
```

**Pro Tip**: The Makefile generated by CubeMX comes with a predefined rule called `clean` to delete all generated files during the build process (object files, binaries, ... in the `build/` directory).
This rule is very useful to force rebuild all or to cleanup the project directory before packaging your project for archiving.
```
$ make clean
rm -fR .dep build
```


4 - Programming the board
-------------------------

### Option 1 - Using STM32CubeProgrammer GUI:
1. Open **STM32CubeProgrammer**
2. Connect a USB cable from the board to your computer
3. Click "**Connect**"
4. Go to the "Erasing & Programming" window
5. **Browse** to load the binary file (`*.hex`, `*.bin` or `*.elf` located in the `build/` directory)
    1. In case of a `*.bin` binary, the Start Address needs to be specified (typically `0x08000000` for STM32)
6. Click **Start Programming**
7. By default, CubeProgrammer does not run the application after programming. **Press the black reset button** to run the firmware. You should see LD2 blinking.

![STM32CubeProgrammer programming](images/CubeProg_program.png)

**Note**: Because STM32CubeProgrammer is still relatively new, chances are you will have to upgrade your ST-Link firmware.

For more information, you can refer to the [STM32CubeProgrammer User Manual](http://www.st.com/resource/en/user_manual/dm00403500.pdf)

### Option 1.1 - Using STM32CubeProgrammer CLI:
Below are some example commands to erase and program the target using STM32CubeProgrammer CLI:
```
/Applications/STM32CubeProgrammer/STM32CubeProgrammer.app/Contents/MacOs/bin/STM32_Programmer_CLI -c port=SWD -e all
/Applications/STM32CubeProgrammer/STM32CubeProgrammer.app/Contents/MacOs/bin/STM32_Programmer_CLI -c port=SWD mode=UR reset=HWrst -e all # hold reset button then release when connecting
/Applications/STM32CubeProgrammer/STM32CubeProgrammer.app/Contents/MacOs/bin/STM32_Programmer_CLI -c port=SWD -w build/Example_Project.elf
```

### Option 2 - Using texane stlink:
If all you want to do is program the board, then run any of the following commands:
```
$ st-flash write ./build/*.bin 0x08000000
$ st-flash --format ihex write ./build/*.hex
```


Otherwise, to program and debug run the gdb server with:
```
$ st-util
```

### Option 3 - Using OpenOCD:
OpenOCD requires a a configuration file. If you installed openOCD using Homebrew, list of provided configuration (`*.cfg`) files can be found using the following command:
```
$ ls /usr/local/Cellar/open-ocd/0.10.0/share/openocd/scripts/board/
$ ls /usr/local/Cellar/open-ocd/0.10.0/share/openocd/scripts/interface/
$ ls /usr/local/Cellar/open-ocd/0.10.0/share/openocd/scripts/target/
```

For example, you could the following command to program and verify using elf/hex/s19 files. Verify, reset and exit are optional parameters. Binary files need the flash address passing.
```
$ openocd -f board/st_nucleo_l476rg.cfg -c "program build/Example_Project.hex verify reset exit"
$ openocd -f interface/stlink-v2-1.cfg -f target/stm32l4x.cfg -c "program build/Example_Project.elf verify reset exit"
$ openocd -f interface/stlink-v2-1.cfg -f target/stm32l4x.cfg -c "program build/Example_Project.bin 0x08000000 verify exit"
```


More examples and documentation available [here](http://openocd.org/doc/html/Flash-Programming.html#Flash-Programming)

5 - Debugging
-------------

VS Code - Cortex Debug plugin settings

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug",
            "cwd": "${workspaceRoot}",
            "executable": "./build/*.elf",
            "request": "launch",
            "type": "cortex-debug",
            "servertype": "openocd",
            "configFiles": [
                "interface/stlink-v2.cfg",
                "target/stm32f1x.cfg",
            ],
            "swoConfig": {
                "enabled": true,
                "swoFrequency": 2000000,
                "cpuFrequency": 8000000,
                "decoders": [
                    {
                        "port": 0,
                        "label": "port0",
                        "showOnStartup": true,
                        "type": "console",
                    }
                ],
            },
        }
    ]
}
```


[STM32CubeMX]:http://www.st.com/en/development-tools/stm32cubemx.html
[UM1718]:https://www.st.com/resource/en/user_manual/dm00104712.pdf
[GNU ARM Embedded Toolchain]:https://developer.arm.com/open-source/gnu-toolchain/gnu-rm
[STM32CubeProgrammer]:http://www.st.com/en/development-tools/stm32cubeprog.html
[Homebrew]:https://brew.sh/
[OpenOCD]:http://openocd.org/
[texane/stlink]:https://github.com/texane/stlink
[NUCLEO-L476RG]:http://www.st.com/en/evaluation-tools/nucleo-l476rg.html
[gdb-dashboard]:https://github.com/cyrus-and/gdb-dashboard

[uart.c]:https://gist.github.com/glegrain/ca92f631e578450a933c67ac3497b4df
