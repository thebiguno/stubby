# Copied from variables.mk
OS=$(shell uname -s)

#AVRDude options
ifeq ($(OS),Linux)
	AVRDUDE_PORT=/dev/ttyUSB0
else
	AVRDUDE_PORT=/dev/tty.usbserial*
endif
DFU=dfu-programmer
AVRDUDE=avrdude
AVRDUDE_PROGRAMMER=stk500v1

OBJCOPY=avr-objcopy
OBJDUMP=avr-objdump
AVRSIZE=avr-size
CFLAGS=-mmcu=$(MMCU) -pedantic -Os -Wall
LFLAGS=-Wl,-u,vfprintf -lprintf_flt -lc

# End of variables.mk

# Most user configurable options are in this file.
include hardware.mk

ifndef PCB_REVISION
	PCB_REVISION=2
endif

ifndef MAGNETOMETER
	MAGNETOMETER=1
	ifndef MAGNETOMETER_ORIENTATION_OFFSET
		MAGNETOMETER_ORIENTATION_OFFSET=3.1415926
	endif
endif

ifndef DISTANCE_SENSOR
	DISTANCE_SENSOR=1
endif

ifndef $(F_CPU)
ifeq '2' '$(PCB_REVISION)'
	F_CPU=20000000
else
	F_CPU=12000000
endif
endif

ifndef TWI_FREQ
	TWI_FREQ=400000L
endif


CDEFS=-DF_CPU=$(F_CPU) -lc -lm -DTWI_FREQ=$(TWI_FREQ) -DPCB_REVISION=$(PCB_REVISION) -DMAGNETOMETER=$(MAGNETOMETER) -DMAGNETOMETER_ORIENTATION_OFFSET=$(MAGNETOMETER_ORIENTATION_OFFSET) -DDISTANCE_SENSOR=$(DISTANCE_SENSOR) -DSERIAL_BUFFER_SIZE=255 -DPWM_MAX_PINS=21
PROJECT=stubby
MMCU=atmega1284p
COMPILER=avr-g++
SOURCES=Stubby.cpp Leg.cpp \
	controllers/calibration.c controllers/processing.c controllers/universal_controller.c \
	gait/gait_tripod.c \
	hardware/battery.c hardware/distance_HCSR04.c hardware/magnetometer_HMC5883L.c  hardware/servo.c hardware/status.c hardware/timer2.c \
	types/Point.cpp \
	util/convert.c util/delays.c util/math.c \
	lib/protocol/protocol.c lib/protocol/protocol_serial.c lib/pwm/pwm.c lib/pwm/pwm.S lib/serial/serial.c lib/serial/serial_async_tx.c lib/twi/twi.c lib/Ring/Ring.cpp
AVRDUDE_PROGRAMMER=usbtiny

LFUSE=0xF7
HFUSE=0xD1
EFUSE=0xFE


# Copied from targets.mk

ifndef COMPILER 
	COMPILER=avr-gcc
endif
ifeq 'avr-gcc' '$(COMPILER)'
	CFLAGS += -std=gnu99
endif

ifeq 'stk500v1' '$(AVRDUDE_PROGRAMMER)'
	ifeq ($(OS),Linux)
		AVRDUDE_PREP_COMMANDS=stty -F $(AVRDUDE_PORT) hupcl
	endif
	AVRDUDE_ARGS += -P $(AVRDUDE_PORT) -b 19200
endif
#We use the arduino programmer to mean 'Arduino Uno with Optiboot'; this is stk500v1 compatible at 115200 baud.
ifeq 'arduino' '$(AVRDUDE_PROGRAMMER)'
	ifeq ($(OS),Linux)
		AVRDUDE_PREP_COMMANDS=stty -F $(AVRDUDE_PORT) hupcl
	endif
	AVRDUDE_ARGS += -P $(AVRDUDE_PORT) -b 115200
endif

ifeq 'usbtiny' '$(AVRDUDE_PROGRAMMER)'
	AVRDUDE_SPEED = -B 1
endif

#If an EFUSE variable has been set, we program the extended fuses too
ifeq '' '$(EFUSE)'
	EXTENDED_FUSE_WRITE=
else
	EXTENDED_FUSE_WRITE=-U efuse:w:$(EFUSE):m
endif

# Default target.
all: clean build

build: $(PROJECT).hex

assembly: $(PROJECT).asm

$(PROJECT).hex: $(PROJECT).out
	$(OBJCOPY) -j .text -j .data -O ihex $(PROJECT).out $(PROJECT).hex
	$(AVRSIZE) -d -C --mcu=$(MMCU) $(PROJECT).out
	@rm -f $(PROJECT).out

$(PROJECT).asm: $(PROJECT).out
	$(OBJDUMP) -C -d $(PROJECT).out

$(PROJECT).out: $(SOURCES) 
	$(COMPILER) $(CDEFS) $(CFLAGS) -I./ -o $(PROJECT).out $(CLIBS) $(SOURCES) $(LFLAGS)


program: all
ifeq 'dfu' '$(AVRDUDE_PROGRAMMER)'
	$(DFU) $(MMCU) erase
	$(DFU) $(MMCU) flash $(PROJECT).hex
else
	$(AVRDUDE_PREP_COMMANDS)
	$(AVRDUDE) -F -p $(MMCU) -c $(AVRDUDE_PROGRAMMER) \
		$(AVRDUDE_ARGS) $(AVRDUDE_SPEED)\
		-U flash:w:$(PROJECT).hex 
endif

fuse:
ifeq 'dfu' '$(AVRDUDE_PROGRAMMER)'
	echo "Cannot set fuses in DFU Programmer mode"
else
	$(AVRDUDE) -F -p $(MMCU) -c $(AVRDUDE_PROGRAMMER) \
		$(AVRDUDE_ARGS) \
		-U lfuse:w:$(LFUSE):m -U hfuse:w:$(HFUSE):m $(EXTENDED_FUSE_WRITE)
endif

readfuse: 
ifeq 'dfu' '$(AVRDUDE_PROGRAMMER)'
	echo "Cannot set fuses in DFU Programmer mode"
else
	$(AVRDUDE) -V -F -p $(MMCU) -c $(AVRDUDE_PROGRAMMER) \
		$(AVRDUDE_ARGS) \
		-U lfuse:r:-:h -U hfuse:r:-:h -U efuse:r:-:h
endif

readeeprom: 
ifeq 'dfu' '$(AVRDUDE_PROGRAMMER)'
	$(DFU) $(MMCU) dump-eeprom --quiet | hexdump
else
	$(AVRDUDE) -V -F -p $(MMCU) -c $(AVRDUDE_PROGRAMMER) \
		$(AVRDUDE_ARGS) \
		-U eeprom:r:-:h
endif

clean:
	rm -f *.o
	rm -f $(PROJECT).hex

#End of targets.mk