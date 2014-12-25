---
layout: post
title: "efm32 an0060 encrypt aes128 support"
---
In AN0060, there is a encrypt.exe which support AES256 only. But for ZG device supports AES128, we need to make a change and generate new exe file.  The steps as follows:

* Download libtom source code
Goto https://github.com/libtom/libtomcrypt, get latest version 1.17.
* Extract it and generate directory "libtomcrypt-1.17"
* Run msys under windows.
Here we use the msys from Railsinstaller directory

```
$ C:\RailsInstaller\DevKit\msys.bat
```

* In msys window, enter libtomcrypt-1.17, edit makefile.
Uncomment line 12 and 13, save and close it.

```
CC=gcc
LD=ld
```

* Generate libtomcrypt.a file

```
$ make
```

The generated lib file in "libtomcrypt-1.17" directory.

* Copy AN0060 encrypt.c into "libtomcrypt-1.17" directory.
The file can be found in C:\SiliconLabs\SimplicityStudio\v2\developer\sdks\efm32\v2\an\an0060_efm32_aes_bootloader\src\encrypt\encrypt.c

* Modify encrypt.c to support AES128, save and close it.
On line 51 change the AES_KEY_SIZE from 32 to 16

```c
-#define AES_KEY_SIZE 32
+#define AES_KEY_SIZE 16
```

* Generate encrypt.exe

```
$ gcc -o encrypt encrypt.c -I./src/headers  -L./ -ltomcrypt
```

The generated encrypt.exe in "libtomcrypt-1.17" directory.
