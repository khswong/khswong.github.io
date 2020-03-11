---
layout: post
title: Port a Keil uVision (.uvproj) project into arm-gcc/make
---

I'm not yet smart enough to wrangle more complicated build systems, and luckily I haven't yet had to. However, I don't like being paywalled behind expensive compilers and IDEs and embedded development "studios" and much rather use open source toolchains as much as possible. Here's some documentation on how to convert a `.uvproj` to something using arm-gcc. 

## Disclosure

Migrating from compiler to compiler _will_ give you different asm, different binaries and therefore don't give you a guarantee that the behavior between systems will be the same. This should be obvious, as the code emitted and linking and optimizations are all different. 

Other than that, Keil's armcc, I think, defaults to linking with [MicroLib](http://www.keil.com/arm/microlib.asp). We are going to be linking with [newlib-nano](https://github.com/32bitmicro/newlib-nano-1.0). 

## Makefile skeleton:

We're going to be trying to port the [DieBieMS firmware](https://github.com/DieBieEngineering/DieBieMS-Firmware/), which uses the STM32F303, an Arm Cortex-M4 processor. Here's the Makefile I used in full:

```
# Subdirectory make stuff
VPATH = Main 

include Libraries/lib.mk
include Modules/modules.mk 
include Device/device.mk
include Drivers/drivers.mk

PROJECT = DieBieBMS-firmware

# Compiler stuff
TRGT = arm-none-eabi-
CC   = $(TRGT)gcc
CPPC = $(TRGT)g++
# Enable loading with g++ only if you need C++ runtime support.
# NOTE: You can use C++ even without C++ support if you are careful. C++
#       runtime support makes code size explode.
LD   = $(TRGT)gcc
#LD   = $(TRGT)g++
CP   = $(TRGT)objcopy
AS   = $(TRGT)gcc -x assembler-with-cpp
AR   = $(TRGT)ar
OD   = $(TRGT)objdump
SZ   = $(TRGT)size
HEX  = $(CP) -O ihex
BIN  = $(CP) -O binary

# Architecture specific stuff - linker script and architecture
LDSCRIPT = STM32F303CCTx_FLASH.ld
MCU  = cortex-m4

# Define C warning options here
CWARN = -Wall -Wextra -Wundef -Wstrict-prototypes -Wshadow
# Define extra C flags here
CFLAGS = --specs=nosys.specs --specs=nano.specs
CFLAGS += -mthumb -mthumb-interwork -mcpu=$(MCU) -Os -mlittle-endian 
CFLAGS += -mfpu=fpv4-sp-d16 -mfloat-abi=hard -march=armv7e-m
CFLAGS += -D ARM_MATH_CM4 -D STM32F303xC -D USE_HAL_DRIVER -D ARMGCC
CFLAGS += -ffunction-sections -fdata-sections
CFLAGS += -lc -lm -std=c99 
CFLAGS += $(CWARN) -g -Wa,-alms=$(LSTDIR)/$(notdir $(<:.c=.lst)) 
LDFLAGS = $(CFLAGS) 
LDFLAGS += -Wl,--script=$(LDSCRIPT),-lc,-lm,-lnosys,-Map=${BUILDDIR}/${PROJECT}.map,--cref,-u,ResetHandler,--gc-sections
ASFLAGS  = $(CFLAGS) -Wa,-alms=$(LSTDIR)/$(notdir $(<:.s=.lst)) 

ASMSRC = $(STARTUPASM)

CSRC := Main/main.c \
		$(HAL_SRC) \
		$(DEVICE_SRC) \
		$(LIB_SRC) \
		$(MOD_SRC) \
		$(HWDRIVER_SRC) \
		$(SWDRIVER_SRC) \

INCDIR = Main \
		$(LIB_INC) \
		$(MOD_INC) \
		$(HWDRIVER_INC) \
		$(SWDRIVER_INC) \
		$(HAL_INC)

IINCDIR   = $(patsubst %,-I%,$(INCDIR))

ASMOBJS   = $(addprefix $(OBJDIR)/, $(notdir $(ASMSRC:.s=.o)))
COBJS    = $(addprefix $(OBJDIR)/, $(notdir $(CSRC:.c=.o)))
OBJS	 = $(ASMOBJS) $(COBJS)

BUILDDIR = build
OBJDIR = $(BUILDDIR)/obj
LSTDIR = $(BUILDDIR)/lst

OUTFILES = $(BUILDDIR)/$(PROJECT).elf \
           $(BUILDDIR)/$(PROJECT).hex \
           $(BUILDDIR)/$(PROJECT).bin \
		   $(BUILDDIR)/$(PROJECT).s \

## Makefile rules

all: $(BUILDDIR) $(OBJDIR) $(LSTDIR) $(OBJS) $(OUTFILES)

$(BUILDDIR):
	mkdir -p $(BUILDDIR)

$(OBJDIR):
	mkdir -p $(OBJDIR)

$(LSTDIR):
	mkdir -p $(LSTDIR)

$(COBJS) : $(OBJDIR)/%.o : %.c Makefile
	@echo Compiling $(<F)
	$(CC) -c $(CFLAGS) $(OPT) -I. $(IINCDIR) $< -o $@

$(ASMOBJS) : $(OBJDIR)/%.o : %.s Makefile
	@echo Compiling $(<F)
	$(AS) -c $(ASFLAGS) $(IINCDIR) $< -o $@

$(OBJS): | $(BUILDDIR) $(OBJDIR) $(LSTDIR) 

%.elf: $(OBJS) $(LDSCRIPT)
	@echo Linking $@
	$(LD) $(OBJS) $(LDFLAGS) -o $@

%.hex: %.elf $(LDSCRIPT)
	@echo Creating $@
	$(HEX) $< $@

%.bin: %.elf $(LDSCRIPT)
	@echo Creating $@
	$(BIN) $< $@

$(BUILDDIR)/$(PROJECT).s : $(BUILDDIR)/$(PROJECT).elf
	@echo Creating $@
	$(OD) -S -d $< > $@

clean:
	rm -rf $(BUILDDIR) 

rebuild: clean all
```

Let's go through this piece by piece. 

## Compiler and compiler flags

Here we set up our toolchain. Nothing special, although you might want to specify in your path which `arm-none-eabi-gcc` you're using. 

```
# Compiler stuff
TRGT = arm-none-eabi-
CC   = $(TRGT)gcc
CPPC = $(TRGT)g++
# Enable loading with g++ only if you need C++ runtime support.
# NOTE: You can use C++ even without C++ support if you are careful. C++
#       runtime support makes code size explode.
LD   = $(TRGT)gcc
#LD   = $(TRGT)g++
CP   = $(TRGT)objcopy
AS   = $(TRGT)gcc -x assembler-with-cpp
AR   = $(TRGT)ar
OD   = $(TRGT)objdump
SZ   = $(TRGT)size
HEX  = $(CP) -O ihex
BIN  = $(CP) -O binary
```

This is just pointing to our linker script and defining which MCU we're using. We'll come back to the linker file, because this is ultimately the part that gave me the most trouble (although this is mostly because I'm illiterate and can't read).

```
# Architecture specific stuff - linker script and architecture
LDSCRIPT = STM32F303CCTx_FLASH.ld
MCU  = cortex-m4
```

Our compiler flags are more complicated. I mostly referred to [this reference sheet](https://www.st.com/content/ccc/resource/technical/document/user_manual/group1/cd/29/43/c5/36/c0/40/bb/Newlib_nano_readme/files/newlib-nano_readme.pdf/jcr:content/translations/en.newlib-nano_readme.pdf) when trying to figure out what flags to use. 

```
# Define C warning options here
CWARN = -Wall -Wextra -Wundef -Wstrict-prototypes -Wshadow
# Define extra C flags here
CFLAGS = --specs=nosys.specs --specs=nano.specs
```

`--specs=nosys.specs` disables semihosting and asks you to retarget stdio over an interface (such as USART, etc...) by providing your own implementation for `putc()`, etc... Semihosting is where `printf()` and other IO functions are being run directly on the host debugging computer, which is enabled by linking `--specs=rdimon.specs`. I advice to disable semihosting where possible. `--specs=nano.specs` tells arm-gcc to link with newlib nano, which allows you to shrink the size of the binary. 

This project compiled with Keil's ARMCC will produce a binary around 83K; compiled with standard newlib, arm-gcc will provide a binary around 115K. The binary size compiled with newlib-nano will be around 85K, which is acceptably close to the Arm compiler.

```
CFLAGS += -mthumb -mthumb-interwork -mcpu=$(MCU) -Os -mlittle-endian 
```

```
CFLAGS += -mfpu=fpv4-sp-d16 -mfloat-abi=hard -march=armv7e-m
CFLAGS += -D ARM_MATH_CM4 -D STM32F303xC -D USE_HAL_DRIVER -D ARMGCC
```
Referring to the architecture options table in the reference sheet, we can enable the hardware floating point instructions and link against those instead of linking against the software floating point implementation. 

`-march=armv7e-m` lets arm-gcc determine which kind of instructions it can emit while generating assembly. It can try to auto-detect, but we might as well be explicit with it.

`-D STM32F303xC -D USE_HAL_DRIVER` are STM32-specific; it lets us use the STM32 HAL (hardware abstraction layer) and to let the HAL know which processor we're using.

I defined `-D ARMGCC` in sections where another developer had added op-codes/assembly that is interpreted by the Arm compiler and not arm-gcc - specifically, `__nop()` is Arm compiler and `__NOP()` is arm-gcc.  

```
CFLAGS += -ffunction-sections -fdata-sections
```

`-ffunction-sections` instructions arm-gcc to create .text sections for each function generated, which can then be pruned for unused functions at link time. `-fdata-sections` is analogous to `-ffunction-sections` but with global/static variables that then get garbage collected at link time. 

```
CFLAGS += -lc -lm -std=c99 
CFLAGS += $(CWARN) -g -Wa,-alms=$(LSTDIR)/$(notdir $(<:.c=.lst)) 
LDFLAGS = $(CFLAGS) 
LDFLAGS += -Wl,--script=$(LDSCRIPT),-lc,-lm,-lnosys,-Map=${BUILDDIR}/${PROJECT}.map,--cref,-u,ResetHandler,--gc-sections
ASFLAGS  = $(CFLAGS) -Wa,-alms=$(LSTDIR)/$(notdir $(<:.s=.lst)) 
```

We actually link with `arm-none-eabi-gcc` and pass linker flags using `-Wl`. `-Wl,--script=$(LDSCRIPT)` passes in our linker script. `-lc,-lm,-lnosys` links the libraries that we want (libc, libmath, libnosys for retarget). `-Map=${BUILDDIR}/${PROJECT}.map,--cref` provides our linker map with a cross-reference table. `-u,ResetHandler` forces the ResetHandler symbol to be undefined to be linked with the startup assembly file provided by STM. `--gc-sections` enables garbage collection, which is only useful with `-ffunction-sections -fdata-sections` as discussed above.

## Input files

Okay, the first tedious part is done. Now it's time to grep some files for what we actually need.

```
# Subdirectory make stuff
VPATH = Main 

include Libraries/lib.mk
include Modules/modules.mk 
include Device/device.mk
include Drivers/drivers.mk
```


