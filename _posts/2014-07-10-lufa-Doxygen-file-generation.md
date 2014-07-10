---
layout: post
title: "lufa doxygen file generation"
---
### 1. Download doxygen1.8.7 from www.doxygen.org.  And install it.

### 2. Add C:\SiliconLabs\SimplicityStudio\v2\developer\msys\1.0\bin in path 

```
$ set path=C:\SiliconLabs\SimplicityStudio\v2\developer\msys\1.0\bin;%path%
```

### 3. Got LUFA root directory, type "make doxygen" in command line.

### 4. Found many error message and fix them in source code. Something like:

```
"warning: multiple use of section label 'Sec_Serial_Dependencies'"
"warning: Member documentation for Endpoint_Discard_Stream found several times in @defgroup groups!"
"warning: argument 'Options' of command @param is not found in the argument list of USB_Init(uint8_t endpoint_desc)"
```

### 5. But there is an error message related with Doxygen version. The Doxyfile is 1.8.6, the Doxygen we are using is v1.8.7

```
Warning: Tag `XML_SCHEMA' at line 1828 of file `-' has become obsolete.  To avoid this warning please remove this line from your configuration file or upgrade it using "doxygen -u" 
Warning: Tag `XML_DTD' at line 1834 of file `-' has become obsolete.       To avoid this warning please remove this line from your configuration file or upgrade it using "doxygen -u
```
Goto directory LUFA\, run "doxygen -u", return to LUFA directory, "make doxygen" again. The LUFA directory warning pass but other directory occurs same error message. Find many Doxyfile in whole directory.  So the simplest way is to comment all XML_SCHEMA and XML_DTD in Doxyfiles. Use SublimeText3 find and replay XML_DTD and XML_SCHEMA in doxyfile under LUFA root direcotry

### 6. An new error comes when generating documentation. 

```
make[4]: Entering directory `./lufa/Demos/Device/LowLevel/EFM32Demos'
make[4]: *** No rule to make target `doxygen'.  Stop.
```
* Prepare a makefile, doxyfile and asf.xml in directory EFM32Demos. Looks like this
* 
```
all:
include ./VCP/makefile
```
* Modify makefile under VCP directory.

```
…
MCU          = EFM32GG990
ARCH         = EFM32GG
BOARD        = DK3750
F_CPU        = 48000000
F_USB        = $(F_CPU)
OPTIMIZATION = s
TARGET       = VCP
SRC          = VirtualSerial.c Descriptors.c $(LUFA_SRC_USB)
LUFA_PATH    = ../../../../LUFA
CC_FLAGS     = -DUSE_LUFA_CONFIG_HEADER -IConfig/
LD_FLAGS     =
…
```
* Also, modify asf.xml as well according EFM32GG information.

### 7. We have an new  error comes out.

```
../../../../LUFA/Build/lufa_build.mk:134: *** Unsupported architecture "EFM32GG".  Stop.
```
* Open lufa\LUFA\Build\lufa_build.mk
Add EFM32GG in it.  Part of modifications looks like

```
…
else ifeq ($(ARCH), EFM32GG)
   CROSS        := gcc
…
```
* An error comes out

```
../../../../LUFA/Build/lufa_atprogram.mk:89: *** Unsupported architecture "EFM32GG".  Stop.
```
* Open LUFA/Build/lufa_atprogram.mk, add EFM32GG in it.

### 8. Got a new error comes out

```
/CMSIS/efm32gg/system_efm32gg.c:7: warning: multiple use of section label 'License', (first occurrence: C:/Users/mading/Dropbox/work/Bitbucket/lufa/Demos/Device/LowLevel/EFM32Demos/VCP/BSP/bsp_dk_3201.c, line 6.
```
Just simply change @section to section in all em_lib files. 

### 9. Execute "make doxygen" and pass. The documentation files are saved in Documentation directory. 
LUFA "make doxygen" operation complete.

### 10. Double clikc on \lufa\LUFA\Documentation\html\index.html, we will see Silabs EFM32 added in it.

### 11. Sometimes we got this error message:
"error: Could not open file globals.html for writing"
Just simply remove all directory "Documentation" under root directory. The reason is unknown. 


