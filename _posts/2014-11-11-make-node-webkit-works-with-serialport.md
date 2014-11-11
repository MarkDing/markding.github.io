---
layout: post
title: "Make node-webkit works with serialport"
---

I have met issue on making serialport works under node-webkit. There are a lot of similar issues happened. So here is the way to solve the problem. Hope that helps.

# Software installation 
The PC environment is Windows 7 64-bit

## Download Node.js 32 bit version.
We got errors when build serialport with node-pre-gyp, it needs 32 bit version. The Node.js version currently used is v0.10.33
http://nodejs.org/dist/v0.10.33/node-v0.10.33-x86.msi

## Download Node-webkit
Only v0.8.6 can works correctly. 
http://dl.node-webkit.org/v0.8.6/node-webkit-v0.8.6-win-ia32.zip

## Install serialport module for Node.js
* Make sure you have python 2.7x installed and added into path environment. 
http://www.python.org/download/releases/2.7.7/

* Install nw-gyp 
Enter directory C:\Users\username\AppData\Roaming\npm

```
$ npm install nw-gyp -g
```

* Install node-pre-gyp
Enter working directory, execute command

```
$ npm install node-pre-gyp -g
```

* Install serialport 
The current serialport version is v1.4.6. Enter working directory and install it.

```
$ cd c:\work\SmartCupApp
$ npm install serialport
```
It installs serialport module under .\node_modules\serialport

* Rebuild serialport with node-webkit config.

```
$ cd node_modules\serialport
$ node-pre-gyp build --runtime=node-webkit --target=0.8.6 
```

https://github.com/rogerwang/node-webkit/wiki/Build-native-modules-with-nw-gyp

* But it still reports cannot find serialport module from the directory when execute script file, Just check current directory name to match with its output message, and then we can open the serial port. 

```
[8732:1110/155118:INFO:CONSOLE(343)] "Uncaught Error: Cannot find module 'C:\work\SmartCupApp\node_modules\serialport\build\serialport\v1.4.6\Release\node-webkit-v11-win32-ia32\serialport.node'", source: module.js (343)
```

Check the directory it generated, it is `node-webkit-v0.8.6-win32-ia32`, change it to `node-webkit-v11-win32-ia32` and the serialport works. 
