# 3. Zigbee OTA Update
First at all, we would like to do simple comparison between current design and new design we propose here. That helps us better understanding what is major changes and how we optimized the OTA procedure. 

Zigbee OTA bootloading session involves two devices, OTA Client device and OTA Server device. Generally, Client is the Zigbee device running on System-on-Chip mode which will request and receive the new firmware image from the Server device. 

And the Zigbee OTA Server can run on either SoC or NCP mode with a host, OTA Server will store the new Client firmware image in the local storage or host's local file system depends on which mode the OTA Server is using.
The following figures show diagram of two different scenarios of Zigbee OTA.
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ncpServer-client.png">  
</div> 
<div align="center">
  <b>NCP-based OTA server and SoC mode Client hardware diagram</b>
</div>  
</br>

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/socServer-client.png">  
</div> 
<div align="center">
  <b>SoC mode OTA server and SoC mode Client hardware diagram</b>
</div>  
</br>

Get reference of detailed introduction of current design at [AN728: Over-the-Air Bootload Server and Client Setup][AN728].

## 3.1. Current OTA process

The existing AN728 show how to configure the NCP-based Server and SoC mode client for OTA process as below.   
* Connect client SoC WSTK device with UART console to PC  
* Manually copy the client OTA image file under OTA server host application Z3Gatewayhost directory  
* Run Z3GatewayHost.exe from PC to connect with NCP WSTK board  
* Execute following commands from OTA server Z3GatewayHost to form and open the network  
  * $net leave  
  * $plugin network-creator start 1  
  * $plugin network-creator-security open-network  
* Execute following command from OTA client to join the network and then start OTA process.   
  * $plugin network-steering start 0  
  * $plugin ota-client start  

  It takes 5.5 minutes to complete the OTA update. 

## 3.2. New OTA process
### 3.2.1. NCP-based Server and SoC Client
* Run Z3GatewayHost.exe to connect with NCP WSTK board. It form and open the network  
* Press any button on client WSTK board. It join the network and start OTA update without any interaction.   

The two LEDs on board show the status of network and OTA update. 

### 3.2.2. SoC Server and SoC Client
* Power on the OTA Server device, it will form and open the network.  
* Store the new client firmware image into the local storage of OTA Server device.  
* Press any button on client WSTK board. It join the network and start OTA update without any interaction.  

The two LEDs on board show the status of network and OTA update.

## 3.3. Implementation

Here we start the detailed steps on implementation to achieve above design idea. 

### 3.3.1. NCP-based Server and SoC Client

#### 3.3.1.1. Zigbee OTA Client

For the design of OTA client. We would like to achieve following functionalities. 
* Press any button on board to start joining network and the OTA update  
* LED1 ON indicates that the device has joined the network. LED OFF is opposite meaning  
* LED0 Blinking indicates the OTA update in progress  

A) Click on "New Project" in Simplicity Studio. Choose "Silicon Labs Zigbee", press Next; Choose "EmberZNet 6.8.0.1 GA SoC 6.8.0.1", press Next; Select "ZigbeeMinimal", press Next; Change the project name as "ZigbeeOTAClient", press Next and then press Finish. 

It open a ZigbeeOTAClient.isc which can config Zigbee related functionalities. There many Tabs on it for configuring different settings of project. 

B) In **ZCL Clusters** tab, Select ZCL device type as **HA devices->HA On/Off Switch**, Check the **Over the Air Bootloading ** client check box from Cluster List pane. The profile ID should be "Home automation (0x0104)"  

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-24-17-26-41.png">
</div> 
</br>

C) In **Printing and CLI** tab, Enable the **Compiled-in** and **Enabled at startup** checkboxes for the **Ota Bootload cluster**   

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-24-16-52-38.png">
</div> 
</br>

D) In **Plugins** tab, Enable following plugins  
* OTA Bootload Cluster Client  
* OTA Bootload Cluster Client Policy  
  *	Change the Firmware version to **0x100**. OTA update only works while the version number changed.   
* OTA Bootload Cluster Common Code  
* OTA Bootload Cluster Storage Common Code  
* OTA Cluster Platform Bootloader  
* OTA Simple Storage Module  
* OTA Simple Storage EEPROM Driver  
  *	Set EEPROM Device Read-modify-write support to "false"  
  * Set Gecko Bootloader Storage Support to "Use first slot"  
* EEPROM  
* Slot Manager  
  

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-24-17-11-23.png">
</div> 
</br>

E) In **Callbacks** tab, enable **Hal Button Isr** because we are going to start the OTA update by pressing a button. Also enable the **Main Init** under "Non-cluster related" and we'll print the version of the running software in this callback function.    

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-24-17-24-12.png">
</div> 
<br>

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ota-client-enable-mainInit.png">
</div> 
</br>

F) In **Includes** tab, add steeringEventControl and its callback steeringEventHandler to manage the joining network operation.  

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-24-17-50-48.png">
</div> 

G) Click on the Generate button on top-right of ZigbeeOTAClient.isc to generate source code of the project

H) Open the ZigbeeOTAClient_callbacks.c and add following function

```c
EmberEventControl steeringEventControl;
/*  
 * LED1 ON if the client device already joined the network. And start the OTA Update  
 * If the client device is not in the network, start joining network process.  
 */  
void steeringEventHandler(void)
{
  emberAfCorePrintln("steeringEventHandler\n\r");
  emberEventControlSetInactive(steeringEventControl);

  if (emberAfNetworkState() == EMBER_JOINED_NETWORK) {
    halSetLed(BOARDLED1);
    otaStartStopClientCommand(true);
  } else {
    EmberStatus status = emberAfPluginNetworkSteeringStart();
    emberAfCorePrintln("%p network %p: 0x%X", "Join", "start", status);
  }
}

/*   
 * LED1 ON while button pressed. LED1 OFF while button released and active   
 * the event steeringEventControl   
 */  
void emberAfHalButtonIsrCallback(int8u button, int8u state)
{
  halSetLed(BOARDLED1);
  if (state == BUTTON_RELEASED) {
    halClearLed(BOARDLED1);
    emberEventControlSetActive(steeringEventControl);
  }
}

/*  
 * This callback is fired when the Network Steering plugin is complete  
 * If the status is success, then LED1 ON, active event steeringEventControl  
 * with 1000 ms delay, start the OTA update in the steeringEventHandler().   
 */  
void emberAfPluginNetworkSteeringCompleteCallback(EmberStatus status,
                                                  uint8_t totalBeacons,
                                                  uint8_t joinAttempts,
                                                  uint8_t finalState)
{
  emberAfCorePrintln("Join network complete: 0x%X", status);  
  if (status == EMBER_SUCCESS) {
    halSetLed(BOARDLED1);
    emberEventControlSetDelayMS(steeringEventControl, 1000);
  }
}

/** @brief Main Init  
 *  
 * This function is called from the application's main function. It gives the  
 * application a chance to do any initialization required at system startup. Any  
 * code that you would normally put into the top of the application's main()  
 * routine should be put into this function. This is called before the clusters,  
 * plugins, and the network are initialized so some functionality is not yet  
 * available.  
        Note: No callback in the Application Framework is
 * associated with resource cleanup. If you are implementing your application on  
 * a Unix host where resource cleanup is a consideration, we expect that you  
 * will use the standard Posix system calls, including the use of atexit() and  
 * handlers for signals such as SIGTERM, SIGINT, SIGCHLD, SIGPIPE and so on. If  
 * you use the signal() function to register your signal handler, please mind  
 * the returned value which may be an Application Framework function. If the  
 * return value is non-null, please make sure that you call the returned  
 * function from your handler to avoid negating the resource cleanup of the  
 * Application Framework itself.  
 *  
 */  
void emberAfMainInitCallback(void)
{
  otaPrintClientInfo();
}
```

I) Build the project and download the firmware image ZigbeeOTAClient.s37 into the client WSTK board. 

If you don't know how to process it. Please get detailed reference at [Download firmware Image][Flash-Image]. After the device boot up, it will output the version of the running software as below which is 0x100.  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ota-client-initial-fw-version.png">
</div> 

J) Generate client OTA image   
We need to have a new client image file for OTA update. Just simply change the **firmware version** in "ZigbeeOTAClient.isc->Plugins->OTA Bootload Cluster Client Policy" to 0x200.   

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-24-18-48-45.png">
</div> 
</br>

Generate the source code and build the project. Copy generated ZigbeeOTAClient.ota file to build/exe/ota-files under Z3GatewayHost project.


#### 3.3.1.2. Zigbee OTA Server in NCP mode with host program

For the design of NCP-based OTA Server. We would like to achieve following functionalities.

* The PC host application Z3GatewayHost.exe automatic form and open the network to let client device join the network  
* Z3GatewayHost.exe start OTA update according the request from client device. (It is default setting of current design)    

A) Click on "New Project" in Simplicity Studio. Choose "Silicon Labs Zigbee", press Next; Choose "EmberZNet 6.8.0.1 GA Host 6.8.0.1", press Next; Select "Z3Gateway", press Next; Keep project name as "Z3GatewayHost" unchanged, press Next and then press Finish.

It open a Z3GatewayHost.isc which can config Zigbee related functionalities. There many Tabs on it for configuring different settings of project. 

The OTA related plugins are enabled by default setting. We need to add several callback functions and event to enable automatic form and open network. 

B) In **Callbacks** tab, enable **Main Init** under "Non-cluster related" to active commissioning event to form or open network. Enable **Complete** of "Network Creator" plugin and **Update Complete** of "OTA Bootloader Cluster Server" under "Plugin-specific callbacks" to active commissioning event to open network after forming network is done and exit the program while OTA update complete.    

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-05-18-17-02-10.png">
</div> 
</br>

C) In **Includes** tab, add **commissioningEventControl** command and **commissioningEventHandler** callback to maintain form and open network.   

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-25-15-22-08.png">
</div> 
</br>

D) Click on the Generate button on top-right of Z3GatewayHost.isc to generate source code of the project

E) Open the Z3GatewayHost_callbacks.c and add following function

```c
EmberEventControl commissioningEventControl;
static uint8_t exit_program = false;
/*  
 * It active the commissioning event with 1000 ms delay  
 */  
void emberAfMainInitCallback(void)
{
  emberEventControlSetDelayMS(commissioningEventControl, 1000);
}
/*  
 * It form network if network doesn't exist. It open the network  
 * if network exist to allow client device to join the network.  
 */  
void commissioningEventHandler(void)
{
  EmberStatus status;

  if(exit_program == true) {
	  exit(0);
  }
  emberAfCorePrintln("commissioningEventHandler\n\r");
  emberEventControlSetInactive(commissioningEventControl);

  status = emberAfNetworkState();
  emberAfCorePrintln("Network state = %d", status);

  if (status == EMBER_NO_NETWORK) {
    status = emberAfPluginNetworkCreatorStart(true);
    emberAfCorePrintln("Form centralized network Start: 0x%X", status);
    return;
  }

  if (status == EMBER_JOINED_NETWORK) {
    status = emberAfPluginNetworkCreatorSecurityOpenNetwork();
    emberAfCorePrintln("Open network: 0x%X", status);
    return;
  }
}

/*  
 * This callback notifies the user that the network creation process has  
 * completed successfully. It activate commissioning event to open the network  
 * with 1000 ms delay.  
 */  
void emberAfPluginNetworkCreatorCompleteCallback(const EmberNetworkParameters *network,  
                                                 bool usedSecondaryChannels)
{
  EmberStatus status;
  emberAfCorePrintln("emberAfPluginNetworkCreatorCompleteCallback");
  emberEventControlSetDelayMS(commissioningEventControl, 1000);
}

/*  
 * This function will be called when an OTA update has finished.  
 */  
void emberAfPluginOtaServerUpdateCompleteCallback(uint16_t manufacturerId,
                                                  uint16_t imageTypeId,
                                                  uint32_t firmwareVersion,
                                                  EmberNodeId source,
                                                  uint8_t status)
{
  emberAfCorePrintln("OTA update completed");
  exit_program = true; // Wait 10 seconds to exit the program
  emberEventControlSetDelayMS(commissioningEventControl, 10000);
}

```

F) Build the Z3GatewayHost project under Cygwin terminal
Enter Z3GatewayHost project directory from Cygwin terminal, execute command
* $make -j8  

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/2020-03-25-16-11-17.png">
</div> 
</br>

The Z3GatewayHost.exe has been generated at ./build/exe/ folder


#### 3.3.1.3. Zigbee OTA Update

* Connect NCP WSTK and Client SoC WSTK to PC  
* Run Z3GatewayHost.exe from PC, make sure new OTA image has been put under ./ota-files folder.   
  * ./build/exe/Z3GatewayHost.exe -p COM5  
* Press any button on Client SoC WSTK start joining network and begin OTA update  
  * The LED1 is ON while joined network. The LED0 keep rapidly blinking during OTA update procedure  
  * The LEDs are OFF while OTA upgrade end response and reboot   
  * Wait one minute for client to complete the image update to application area.   

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ota-server-finish-ota.png">
</div> 

* Reset the Client and you can get the updated firmware version information.  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/ota-client-updated-fw-version.png">
</div> 

### 3.3.2. SoC Server and SoC Client
#### 3.3.2.1. Zigbee OTA Client
From the OTA Client viewpoint, the overall OTA process is the same in both the remote OTA server is SoC mode or NCP mode with host program. The process for OTA Client setup here is identical as section [3.3.1.1 Zigbee OTA Client](#3311-zigbee-ota-client).

#### 3.3.2.2. Zigbee OTA Server in SoC mode

Different with the OTA Server in NCP mode with host program, the OTA Server can be setup on a system-on-chip(SoC) system. For the design of OTA Server in SoC mode, We would like to achieve following functionalities.
* The SoC Server automatic form and open the network to let client device join the network.  
* OTA Server in SoC mode start OTA update according the request from client device. (It is default setting of current design)  

A) Click on "New Project" in Simplicity Studio. Choose "Silicon Labs Zigbee", press Next; Choose "EmberZNet 6.8.0.1 GA SoC 6.8.0.1", press Next; Select "ZigbeeMinimal", press Next; Change the project name as "ZigbeeOTAServer", press Next and then press Finish.

It opens a ZigbeeOTAServer.isc which can config Zigbee related functionalities. There many Tabs on it for configuring different settings of project.   

B) In **ZCL Clusters** tab, Select ZCL device type as HA devices->HA On/Off Light, Check the **Over the Air Bootloading** Server check box from Cluster List pane. The profile ID should be "Home automation (0x0104)"  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-cluster.png">
</div> 
</br>

C) In **Zigbee Stack** tab, set the "Zigbee Device Type" as Coordinator or Router.  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-devicetype.png">
</div> 
</br>

D) In Printing and CLI tab, Enable the "Compiled-in" and "Enabled at startup" checkboxes for the Ota Bootload cluster, and also enable the "Include command and argument descriptions in the embedded code" checkbox.
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-printCLI.png">
</div> 
</br>

E) In **HAL** tab, keep the bootloader type as the default "Application".  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-hal.png">
</div> 
</br>

F) In **Plugins** tab, Enable following plugins  
* Network Creator  
* Network Creator Security  
* Security Link Keys Library  
* Source Route Library     
</br>

* OTA Bootload Cluster Common Code  
* OTA Bootload Cluster Server  
* OTA Bootload Cluster Server Policy  
* OTA Bootload Cluster Storage Common Code  
* OTA Simple Storage EEPROM Driver  
  * Uncheck the option labeled "SOC Bootloading Support"  
  * Set the OTA Storage Start Offset as 540672  
  * Set the OTA Storage End Offset as 540672 + 458752 = 999424  
  * Set EEPROM Device Read-modify-write support to "false"     
**Note:** The parameter "OTA Storage Start Offset" and "OTA Storage End Offset" mean the offset used in storage, here they mean the offset in internal flash. We just align this setting with the default slot start address and size setting in the [gecko bootloader project](#23-bootloader-requirement).     
* OTA Simple Storage Module  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-plugin.png">
</div> 
</br>

G) In Callbacks tab, enable Main Init under "Non-cluster related" to active commissioning event to form or open network.  
Enable Complete of "Network Creator" plugin and Update Complete of "OTA Bootloader Cluster Server" under "Plugin-specific callbacks" to active commissioning event to open network after forming network is done and output some information while OTA update complete.

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-callback-mainInit.png">
</div> 
</br>

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-callback-complete.png">
</div> 
</br>

H) In **Includes** tab, add commissioningEventControl event and commissioningEventHandler callback to maintain form and open network.    
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-setup-include.png">
</div> 
</br>

I) Click on the Generate button on top-right of ZigbeeOTAServer.isc to generate source code of the project.   

J) Modify function "emAfOtaStorageDriverGetRealOffset()" in ota-storage-simple-eeprom\ota-storage-eeprom.c   
A Warning message will pop-up to remind that you are editing a shared file in the SDK. Click the "Make a Copy", and continue to edit the copy of the source file.
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-edit-warning.png">
</div> 
</br>

```c
bool emAfOtaStorageDriverGetRealOffset(uint32_t* offset,  
                                       uint32_t* length)  
{
  *offset += gOtaImageInfoStart;  
  return false;
}  
```

K) Open the ZigbeeOTAServer_callbacks.c and add following function

```c
EmberEventControl commissioningEventControl;

/*  
 * It active the commissioning event with 1000 ms delay  
 */  
void emberAfMainInitCallback(void)
{
  emberEventControlSetDelayMS(commissioningEventControl, 1000);
}
/*  
 * It form network if network doesn't exist. It open the network  
 * if network exist to allow client device to join the network.  
 */  
void commissioningEventHandler(void)
{
  EmberStatus status;

  emberAfCorePrintln("commissioningEventHandler\n\r");
  emberEventControlSetInactive(commissioningEventControl);

  status = emberAfNetworkState();
  emberAfCorePrintln("Network state = %d", status);

  if (status == EMBER_NO_NETWORK) {
    status = emberAfPluginNetworkCreatorStart(true);
    emberAfCorePrintln("Form centralized network Start: 0x%X", status);
    return;
  }

  if (status == EMBER_JOINED_NETWORK) {
    status = emberAfPluginNetworkCreatorSecurityOpenNetwork();
    emberAfCorePrintln("Open network: 0x%X", status);
    return;
  }
}

/*  
 * This callback notifies the user that the network creation process has  
 * completed successfully. It activate commissioning event to open the network  
 * with 1000 ms delay.  
 */  
void emberAfPluginNetworkCreatorCompleteCallback(const EmberNetworkParameters *network,  
                                                 bool usedSecondaryChannels)
{
  EmberStatus status;
  emberAfCorePrintln("emberAfPluginNetworkCreatorCompleteCallback");
  emberEventControlSetDelayMS(commissioningEventControl, 1000);
}

/*  
 * This function will be called when an OTA update has finished.  
 */  
void emberAfPluginOtaServerUpdateCompleteCallback(uint16_t manufacturerId,
                                                  uint16_t imageTypeId,
                                                  uint32_t firmwareVersion,
                                                  EmberNodeId source,
                                                  uint8_t status)
{
  emberAfCorePrintln("OTA update completed");
}
```

L) Build the project and download the firmware image ZigbeeOTAServer.s37 into the OTA Server device.

#### 3.3.2.3. Zigbee OTA Update
* Store the new Client firmware image into the internal flash of OTA Server device with commander.     
**Note:** We need to rename the new Client firmware image with a *.bin extension. The flash starts at 0x84000 with 448kB length is reserved for OTA Storage as illustrated below.     

```
commander.exe flash .\ZigbeeOTAClient.ota.bin --address 0x84000 --serialno 440056128
```

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-flash-layout.png">
</div> 

* Reset the OTA Server device, it will form and open network automatically.  
* Press any button on SoC Client WSTK start joining network and begin OTA update  
  * The LED1 is ON while joined network. The LED0 keep rapidly blinking during OTA update procedure  
  * The LEDs are OFF while OTA upgrade end response and reboot   
  * Wait one minute for client to complete the image update to application area.   

<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-OTA-process-server.png">
</div> 
</br>

* The Client will be reset and you can get the updated firmware version information.  
<div align="center">
  <img src="https://markding.github.io/doc4zhihu/data/files/CM-IoT-OTA-Update/zb-otaServer-socMode-OTA-process-client.png">
</div> 
