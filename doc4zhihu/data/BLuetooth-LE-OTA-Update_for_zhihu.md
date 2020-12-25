# 5. Bluetooth LE OTA Update

In this section, we are going to introduce the current BLE OTA Update method also propose a new implementation here to optimize the OTA Update procedure.

BLE OTA Update procedure consists of two parts, OTA Client and OTA server. Generally, BLE OTA Client running on System-on-Chip mode which receive new firmware image from the Server device. And BLE OTA Server running on either SoC or NCP mode with a host, which will store new firmware image in the local storage or host’s file system and provide new firmware image to the Client device.

Get reference of current Bluetooth OTA detailed introduction at [AN1086: Using the Gecko Bootloader with the Silicon Labs Bluetooth Applications](https://www.silabs.com/documents/public/application-notes/an1086-gecko-bootloader-bluetooth.pdf).

## 5.1 Current OTA process

The existing AN1086 show how to configure the NCP-based Server and SoC mode client for OTA process as below.
* Power on OTA Client SoC device  
* Connect NCP board to PC    
* Run ota-dfu.exe from PC and execute the following command to connect with NCP board via BGAPI protocol and start OTA process to the remote Client device  
  * ./ota-dfu.exe COM5 115200 application.gbl D0:CF:5E:68:A6:95  
  
## 5.2 New OTA process

### 5.2.1 SoC Server and SoC Client

* Power on OTA Server and OTA Client SoC device.  
* Store the new client firmware image into the local storage of OTA Server device.  
* Set up connection between two devices.  
* Press button0 on OTA Client board to send OTA Update request.  
* OTA Server device will receive the OTA request and send new client firmware image to the OTA Client device.  
* OTA Client device receive the image and finish OTA Update automatically.  

LED on board shows the status of OTA Update.

### 5.2.2 NCP-based Server and SoC Client
The proposed new OTA process provide method to Update both Client SoC device and Server NCP device.

* Power on OTA Client SoC device.  
* Connect NCP board to PC.  
* Run ota_uart_dfu.exe from PC and execute the following command to connect with NCP board via BGAPI protocol and start OTA process to the remote Client device.  
  * ./ota_uart_dfu.exe ota COM5 115200 application.gbl D0:CF:5E:68:A6:95  
* Run ota_uart_dfu.exe from PC and execute the following command to connect with NCP board via **UART XMODEM Bootloader** and start NCP device update process.  
  * ./ota_uart_dfu.exe uart COM5 115200 full.gbl   
  

## 5.3 Implementation
Here we start the detailed steps on implementation to achieve above design idea.

### 5.3.1 SoC Server and SoC Client

For the design of SoC-based OTA Server. We would like to send the new firmware image only when receiving OTA request from the OTA Client.
BLE SoC OTA procedure will be shown as the figure below.

  <div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_soc_ota.png">  
  </div>  
  </br> 

#### 5.3.1.1 Internal Flash Bootloader

Both SoC Client and SoC Server need Bootloader for development. Here we use Internal Flash Bootloader for both.

A) Click on "EXAMPLE PROJECT" in Launcher perspective in Simplicity Studio. Choose "Bootloader" to select for "Internal Storage Bootloader (single image on 512kB device)". Choose CREATE, press FINISH.

B) Click on the Generate button on top-right of bootloader-storage-internal-single-512k.isc to generate source code of the project.

C) Save and build the project. Download the firmware image bootloader-storage-internal-single-512k.s37 into both SoC OTA Client and OTA Server board.

#### 5.3.1.2 BLE OTA Client

For the design of BLE OTA client, we would like to achieve following functionalities.
* Press button0 on board to start OTA Update request  
* LED0 ON indicates that OTA Client begins sending OTA request and the OTA Update is in progress  
* LED0 OFF indicates that OTA Client finish receiving image  

A) Click on "Create New Project" in Simplicity Studio. Choose "Bluetooth" and select "Bluetooth - SoC Empty", press "Next". Rename the project with "soc_empty_CLIENT" and then press Finish.

It open gatt_configuration.btconf in which we can config BLE related service and characteristics.

B) The default OTA DFU Service is unconfigurable and it use Bluetooth Apploader to do OTA DFU. For the design of OTA Client, we would like to Remove Apploader area to save memory and implement OTA Firmware Update in User Application.

  <div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_default_ota_service.png">  
  </div>  
  </br> 

In order to remove the apploader defined by default, open "soc_empty_CLIENT.slcp" and then select to open "SOFTWARE COMPONENTS" tab, filter "OTA DFU" in search texbox. Choose "Uninstall"

  <div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_uninstall_ota_dfu.png">  
  </div>  
  </br>  

After uninstalling the OTA DFU component, Silicon Labs OTA (Contributed Item in GATT Configurator) is disappeared. Next, we need to create configurable custom service and characteristics.  

C) Open "gatt_configuration.btconf", you can directly download and import the attached GATT database [gatt_configuration.bgconf](files/CM-IoT-OTA-Update/bluetooth/src/gatt_configuration.btconf) into the project.

Or you can follow steps below to create one by one the service and characteristic.

1. Click "Toggle - Add Standard Bluetooth GATT items - view" in the upper left tab, select "Services" and add "Silicon Labs OTA" Service.
2. Select "Silicon Labs OTA Control" Characteristic. 
   * set its type to "user" instead of "hex".   
   * select "Write", "Write without response" and "Indicate" properties.  
   
  <div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_add_ota_control.png">  
  </div>  
  </br>  

3. Select "Silicon Labs OTA" Service, click "Add new item" in the upper left tab and select "New Characteristic".
   * Rename this new characteristic to "Silicon Labs OTA Data".   
   * Click the ID below and type in "ota_data".   
   * Set its type to "user".  
   * Select "Write" and "Write without response" properties.   
   * Also specify the UUID value of the characteristic as defined by Silabs rules itself as 984227F3-34FC-4045-A5D0-2C581F81A153  
   
  <div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_add_ota_data.png">  
  </div>  
  </br>  

**Note：** If you create OTA service and characteristic yourself, please remember to specify the UUID of service and characteristic as documented in section "3.4 Silicon Labs OTA GATT service" of AN1086. The service and characteristic content and the UUID value are fixed and must not be changed.  
Otherwise, OTA Server can not set up connection with OTA Client.

D) Now your GATT database should look like this.

**Note:** Need to open the **Advertise service** for Silicon Labs OTA service so that the OTA Server can find and connect with it.  

  <div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_add_ota_service.png">  
  </div>  
  </br>  


E) Go back to "soc_empty_CLIENT.slcp", some more components need to be added.

1. To enable the usage of button, filter "button" then Install "Simple Button" as "btn0".
2. To enable the usage of led, filter "led" and Install "Simple Led" as "led0".
3. To enable the usage of debug printing, filter and Install "IO Stream", "Log" and "Tiny printf" related components. 

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_io_stream.png">  
</div>  
</br>

F) Copy the attached [app.c](files/CM-IoT-OTA-Update/bluetooth/src/app.c) file to the project (overwrites the default app.c).

G) Save. Build the project and flash it to your target board.

H) We need to have a new client image file for OTA update. Just simply change the Device name in "gatt_configuration.btconf" -> "Device name" -> Value settings -> Initial value to "OTA Example" also change value length to "11"

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_create_ota_dfu_image.png">  
</div>  
</br>  

I) Save. Rebuild the project.

J) Then, generate new firmware image by clicking "create_bl_files.bat". The .gbl files will be created automatically in to "output_gbl" folder in the project. Rename the generated “application.gbl” to “application.bin” so as to flash it to OTA Server later.


#### 5.3.1.3 BLE OTA Server

For the design of BLE OTA server, we would like to achieve following functionalities.
* LED0 ON indicates that the device has received OTA Update request  
* LED0 OFF indicates that OTA Server finish sending image  

A) Click on "Create New Project" in Simplicity Studio. Choose "Bluetooth" and select "Bluetooth - SoC Empty", press "Next". Rename the project with "soc_empty_SERVER" and then press Finish.

B) Open SOFTWARE COMPONENTS in the project, to enable the usage of led, filter "led" and Install "Simple Led" as "led0".

C) To enable the usage of debug printing, filter and Install "IO Stream", "Log" and "Tiny printf" related components.

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_io_stream.png">  
</div>  
</br> 


D) Open app.c, copy the attached [soc-server-app.c](files/CM-IoT-OTA-Update/bluetooth/src/soc-server-app.c) file to the project (overwrites the default app.c).

E) Build the project and flash it to your target.

#### 5.3.1.4 BLE OTA Update

* Store the new Client firmware image into the internal flash of OTA Server device with commander.  
**Note:** We need to rename the new Client firmware image with a *.bin extention. BG22 flash starts at 0x44000 with 192kB length is reserved for OTA storage as illustrated below.  
```
commander flash application.bin --address 0x44000 --serailno 440179535
```

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_bg22_flashmap.png">  
</div>  
</br> 

* Launch Console for these two devices in Simplicity Studio.  
* Press BTN0 in  OTA Client to start OTA DFU Request.  

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_soc_ota_dfu.png">  
</div>  
</br> 

* OTA Server receives the OTA DFU Request and begin to send new firmware image.  
* OTA Client will receive new image and finish DFU automatically.   
* Use **EFR Connect** to check SoC Client device advertising with new name.  
  
<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_soc_ota_result.png">  
</div>  
</br> 



### 5.3.2 NCP-based Server and SoC Client

For the design of NCP-based OTA Server. We would like to achieve the following functionalities.
* The PC host application ota_uart_dfu.exe start OTA update for remote OTA Client device (default setting of current design).  
* The PC host application ota_uart_dfu.exe start UART DFU update for NCP device via **XModem bootloader** instead of BGAPI bootloader.  

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_ncp_ota.png">  
</div>  
</br>

#### 5.3.2.1 BLE OTA Server Host design

The host application running on PC communicates with NCP target via a virtual serial port connection. Current OTA host example is found in the following directory.

```C:\SiliconLabs\SimplicityStudio\v5\developer\sdks\gecko_sdk_suite\v3.0\app\bluetooth\examples_host\ota_dfu```

The project folder contains a makefile that allows the program to be built using for example MinGW (by running mingw32-make) or Cygwin (by running make). After successful compilation, the executable named **ota-dfu.exe** is placed in subfolder named exe.  

The current design provide only SoC Client OTA Update, in order to vary its function to do NCP DFU via XModem Bootloader. We need to do some change in the current main.c file.

Here, we assume that you've successfully download and install Cygwin so that we can use it directly.

A) Open the following directory, and create an empty file folder named "ota_uart_dfu".
```C:\SiliconLabs\SimplicityStudio\v5\developer\sdks\gecko_sdk_suite\v3.0\app\bluetooth\example_host```

B) Download and copy the attached [main.c](files/CM-IoT-OTA-Update/bluetooth/src/main.c) and [makefile](files/CM-IoT-OTA-Update/bluetooth/src/makefile) file to this folder.

D) Open Cygwin in current directory
```C:\SiliconLabs\SimplicityStudio\v5\developer\sdks\gecko_sdk_suite\v3.0\app\bluetooth\example_host\ota_uart_dfu```

E) Running **make** to get executable file name **ota_uart_dfu.exe** under subfolder /ota_uart_dfu/exe  

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_make_ncp_host.png">  
</div> 
</br>

#### 5.3.2.2 BLE OTA Server NCP design

The development kit that is used as NCP target should be programmed with bootloader as well as NCP application.

A) Click on "EXAMPLE PROJECT" in Launcher perspective in Simplicity Studio. Choose "Bootloader" to select for **UART XMODEM Bootloader**. Choose CREATE, press FINISH.  

It open a bootloader-uart-xmodem.isc in which we can config XModem Bootloader related functionalities. There are many Tabs on if for configuring different settings of project.

B) Click on the Generate button on top-right of bootloader-uart-xmodem.isc to generate source code of the project.

C) Save and build the project. Download the firmware image bootloader-uart-xmodem.s37 into the NCP board.

D) Go back into "EXAMPLE PROJECT" in Laucher perspective, create **Bluetooth - NCP Empty** project.  

E) Build the project and download the firmware image ncp_empty.s37 into the NCP board.

F) We need to have a new NCP image file for NCP update. Just simply change the Device name in "gatt_configuration.btconf" -> "Device name" -> Value settings -> Initial value to "NCP Example" also change value length to "11".

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_create_ncp_dfu_image.png">  
</div>  
</br>

Generate the source code and build the project. Click **create_bl_files.bat** in the project, it will generate .gbl files in "output_gbl" folder. Copy the generated full.gbl to the folder below.  
```C:\SiliconLabs\SimplicityStudio\v5\developer\sdks\gecko_sdk_suite\v3.0\app\bluetooth\example_host\ota_uart_dfu\exe```

**Note:** If you are using Gecko SDK Suite v3.x, please make sure that you've defined two environmental variables, PATH_SCMD and PATH_GCCARM before running the script to generate upgrade images, please see chapter [2.3 Creating Upgrade Images for the Bluetooth NCP Application](https://www.silabs.com/documents/public/application-notes/an1086-gecko-bootloader-bluetooth.pdf) of AN1086 for more information.  

#### 5.3.2.3 BLE OTA Client

From the OTA Client viewpoint, the overall OTA process performs the same behavior in both SoC mode OTA Server and NCP mode OTA Server with host program. The process for OTA Client setup here is identical as section [5.3.1.2 BLE OTA Client](#5312-ble-ota-client).

Also generate OTA update image by Clicking **create_bl_files.bat** in the project, and copy the generated application.gbl to the folder below.  
```C:\SiliconLabs\SimplicityStudio\v5\developer\sdks\gecko_sdk_suite\v3.0\app\bluetooth\example_host\ota_uart_dfu\exe```

BLE OTA Client is the target device to be upgraded over-the-air. It is identified by its Bluetooth address. The Bluetooth address can be found in Simplicity Commander in Unique ID after connecting the target.

For example, the bluetooth address for the given device below is 84:2E:14:31:BA:49.

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_bluetooth_address.png" width="600">  
</div>  
</br>

#### 5.3.2.4 BLE OTA Update

* Connect NCP device to PC and also power on the OTA Client device  
* Run ota_uart_dfu.exe from PC, make sure new OTA iamge has been put under ota-ncp-dfu/exe folder  
  * ./ota_uart_dfu.exe ota COM5 115200 application.gbl xx:xx:xx:xx:xx:xx  

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_ncp_ota_dfu.png">  
</div>  
</br>

* Use **EFR Connect** to check SoC Client device advertising with new name. NCP device originally advertise by "Empty Example". After updating, it advertises with new name "OTA Example".  
  
<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_ncp_ota_result.png">  
</div>  
</br>
  
* Run ota_uart_dfu.exe from PC, make sure new ncp iamge has been put under ota-ncp-dfu/exe folder  
  * ./ota_uart_dfu.exe uart COM5 115200 full.gbl  

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_ncp_uart_dfu1.jpg">  
</div>  
</br>

After initializing NCP Host, a Menu shows up for selecting the following procedure. Typing '1' to choose "upload gbl" to begin uploading NCP Update image to NCP device.

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_ncp_uart_dfu2.png">  
</div>  
</br>

After finishing Uploading, the Menu shows up again. In this time, typing '2' for sending command to reboot the NCP device and run with the new image. This process will finish the NCP Device Firmware Update process. 

* Use **BG Tool** to advertise in NCP device. Connect with the NCP target and click "Create Set" to create advertisement set, then click "Start" to begin advertising.  
  
<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_BGTOOL.png" width = "600">  
</div>  
</br>

* Then use **EFR Connect** to check NCP device advertising with new name. NCP device originally advertise by "Silabs Example". After updating, it advertises with new name "NCP Example".  

<div align="center">
<img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ble_ncp_update_result.png">  
</div>  
</br>
