
# Deadline Sunday 17/04/2022 

## Code Examples

### 1.0 BLEBatteryClient
```
    BLE.configScan()->startScan(2000);    // Scan for 2 seconds, then stop
    BLE.configConnection()->connect(targetDevice, 2000);
    delay(2000);
    connID = BLE.configConnection()->getConnId(targetDevice);

    if (!BLE.connected(connID)) {
        Serial.println("BLE not connected");
    } else {
        BLE.configClient();
        client = BLE.addClient(connID);
        client->discoverServices();
        Serial.print("Discovering services of connected device");
        do {
            Serial.print(".");
            delay(1000);
        } while (!(client->discoveryDone()));
        Serial.println();

        battService = client->getService(BATT_SERVICE_UUID);
        if (battService != nullptr) {
            battChar = battService->getCharacteristic(BATT_LEVEL_UUID);
            if (battChar != nullptr) {
                Serial.println("Battery level characteristic found");
                battChar->setNotifyCallback(notificationCB);
            }
        }
    }
}

void loop() {
    if (BLE.connected(connID)) {
        delay(5000);
        Serial.print("Battery level read: ");
        Serial.println(battChar->readData8());

        delay(5000);
        notifyState = !notifyState;
        if (notifyState) {
            Serial.println("Notifications Enabled");
            battChar->enableNotifyIndicate();
        } else {
            Serial.println("Notifications Disabled");
            battChar->disableNotifyIndicate();
        }
    } else {
        Serial.println("BLE not connected");
        delay(5000);
    }
}
```
#### Comment

- BLEClient is used to create a client object to discover services and characteristics on the connected device.
- setNotifyCallback() is used to register a function that will be called when a battery level notification is received.
- BLE.configClient() is used to configure the Bluetooth stack for client operation.
- addClient(connID) creates a new BLEClient object that corresponds to the connected device.

#

### 1.2 BLEBatteryService

```
#include "BLEDevice.h"

#define BATT_SERVICE_UUID   "180F"

#define BATT_LEVEL_UUID     "2A19"


BLEService battService(BATT_SERVICE_UUID);
BLECharacteristic battChar(BATT_LEVEL_UUID);
BLEAdvertData advdata;
BLEAdvertData scandata;
uint8_t battlevel = 0;
bool notify = false;


void readCB (BLECharacteristic* chr, uint8_t connID) {
    printf("Characteristic %s read by connection %d \n", chr->getUUID().str(), connID);
    chr->writeData8(90);
}


void notifCB (BLECharacteristic* chr, uint8_t connID, uint16_t cccd) {
    if (cccd & GATT_CLIENT_CHAR_CONFIG_NOTIFY) {
        printf("Notifications enabled on Characteristic %s for connection %d \n", chr->getUUID().str(), connID);
        notify = true;
    } else {
        printf("Notifications disabled on Characteristic %s for connection %d \n", chr->getUUID().str(), connID);
        notify = false;
    }
}


void setup() {
    Serial.begin(115200);

    advdata.addFlags();
    advdata.addCompleteName("AMEBA_BLE_DEV");
    scandata.addCompleteServices(BLEUUID("180F"));

    battChar.setReadProperty(true);
    battChar.setReadPermissions(GATT_PERM_READ);
    battChar.setReadCallback(readCB);
    battChar.setNotifyProperty(true);
    battChar.setCCCDCallback(notifCB);

    battService.addCharacteristic(battChar);
    
    BLE.init();
    BLE.configAdvert()->setAdvData(advdata);
    BLE.configAdvert()->setScanRspData(scandata);
    BLE.configServer(1);
    BLE.addService(battService);

    BLE.beginPeripheral();
}


void loop() {
    battlevel = (battlevel + 1)%100;
    battChar.writeData8(battlevel);
    if (BLE.connected(0) && notify) {
        battChar.notify(0);
    }
    Serial.println(battlevel);
    delay(1000);
}
```
#### Comment

- BLEService and BLECharacteristic classes are used to create and define the battery service to run on the Bluetooth device.
- BLE.configAdvert()->setAdvType(GAP_ADTYPE_ADV_IND) is used to set the advertisement type to a general undirected advertisement that allows for connections.
- setReadCallback() and setCCCDCallback() is used to register functions that will be called when the battery level data is read, or notification is enabled by the user.
- BLE.configServer(1) is used to tell the Bluetooth stack that there will be one service running.
- addService() registers the battery service to the Bluetooth stack.

#

### 1.3 BLEBeacon
```

#include "BLEDevice.h"
#include "BLEBeacon.h"

iBeacon beacon;
altBeacon beacon;

- See the following for generating UUIDs:
 https://www.uuidgenerator.net/
- define UUID "00112233-4455-6677-8899-AABBCCDDEEFF"


void setup() {
    // For all possible BLE manufacturer IDs, refer to:
    // www.bluetooth.com/specifications/assigned-numbers/company-identifiers/

    beacon.setManufacturerId(0x004C); // MfgId (0x004C: Apple Inc)
    beacon.setRSSI(0xBF);             // rssi: (0xBF: -65 dBm)
    beacon.setMajor(0x007B);          // 123
    beacon.setMinor(0x01C8);          //456
    beacon.setUUID(UUID);

    BLE.init();
    BLE.configAdvert()->setAdvType(GAP_ADTYPE_ADV_NONCONN_IND);
    BLE.configAdvert()->setAdvData(beacon.getAdvData(), beacon.advDataSize);
    BLE.configAdvert()->setScanRspData(beacon.getScanRsp(), beacon.scanRspSize);
    BLE.beginPeripheral();
}

void loop() {
    delay(1000);
}
```
#### Comment

- setRssi() is used to set the received signal strength indicator (rssi) data field for a beacon. The specification states that this should be the received signal strength from the beacon at a 1 meter distance. With no method to measure this, it is set to -65dBm as an estimate.
- setMajor() andsetMinor() are used to set the two data fields. The purpose of these data are left for the manufacturer of the beacon to define, and can be used in any way.
- setUUID() is used to give the beacon a universally unique identifier (UUID). This is a 128-bit number usually expressed as a hexadecimal string. It is used to identify each unique beacon, and can be randomly generated for free online.
- The BLEBeacon library includes both iBeacon and AltBeacon classes, replace line 6 iBeacon with altBeacon to create an AltBeacon instead. The data fields are mostly the same, with only minor changes, please look at the header files for more details.
- BLE.init() is used to allocate memory and prepare Ameba for starting the Bluetooth stack.
- BLE.configAdvert() is used to configure the Bluetooth advertisement settings, to which we pass the beacon data and set the device as non-connectable.
- BLE.beginPeripheral() starts Ameba in Bluetooth peripheral mode, after which it will begin to advertise with the beacon data provided.

#

### 1.4 BLEScan
```
#include "BLEDevice.h"

int dataCount = 0;

BLEAdvertData foundDevice;

void scanFunction(T_LE_CB_DATA* p_data) {
    Serial.print("Scan Data ");
    Serial.println(++dataCount);
    BLE.configScan()->printScanInfo(p_data);

    foundDevice.parseScanInfo(p_data);
    if (foundDevice.hasName()) {
        if (foundDevice.getName() == String("AMEBA_BLE_DEV")) {
            Serial.print("Found Ameba BLE Device at address ");
            Serial.println(foundDevice.getAddr().str());
        }
    }
    uint8_t serviceCount = foundDevice.getServiceCount();
    if (serviceCount > 0) {
    BLEUUID* serviceList = foundDevice.getServiceList();
        for (uint8_t i = 0; i < serviceCount; i++) {
            if (serviceList[i] == BLEUUID("180F")) {
            Serial.print("Found Battery Service at address ");
            Serial.println(foundDevice.getAddr().str());
            }
        }
    }
}

void setup() {
    Serial.begin(115200);
    BLE.init();
    BLE.configScan()->setScanMode(GAP_SCAN_MODE_ACTIVE);    // Active mode requests for scan response packets
    BLE.configScan()->setScanInterval(500);   // Start a scan every 500ms
    BLE.configScan()->setScanWindow(250);     // Each scan lasts for 250ms
    BLE.configScan()->updateScanParams();
    // Provide a callback function to process scan data.
    // If no function is provided, default BLEScan::printScanInfo is used
    BLE.setScanCallback(scanFunction);
    BLE.beginCentral(0);

    BLE.configScan()->startScan(5000);    // Repeat scans for 5 seconds, then stop
}

void loop() {
    delay(1000);
}
```
#### Comment
- setScanMode(GAP_SCAN_MODE_ACTIVE) is used to set the scan mode. Active scanning will request for an additional scan response data packet from a device when it is found. Passive scanning will only look at the advertisement data, and not request for additional data.
- setScanInterval() and setScanWindow() are used to set the frequency and duration of scans in milliseconds. A scan will start every interval duration, and each scan will last for the scan window duration. The scan window duration should be lesser or equal to the scan interval. Set a short interval to discover devices rapidly, set a long interval to conserve power.
- setScanCallback(scanFunction) is used to register a function to be called when scan results are received. This can be used to set a user function for additional processing of scan data, such as looking for a specific device. If no function is registered, the scan results are formatted and printed to the serial monitor by default.
- beginCentral(0) is used to start the Bluetooth stack in Central mode. The argument 0 is used to indicate that no clients will be operating in central mode.
- startScan(5000) is used to start the scanning process for a specified duration of 5000 milliseconds. The scan will repeat according to the set scan interval and scan window values. After 5000 milliseconds, the scan process will stop, and will be ready to be started again.

#

### 1.5 BLEUartClient

```
#include "BLEDevice.h"

#define UART_SERVICE_UUID      "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHARACTERISTIC_UUID_RX "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHARACTERISTIC_UUID_TX "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"

#define STRING_BUF_SIZE 100

BLEAdvertData foundDevice;
BLEAdvertData targetDevice;
BLEClient* client;
BLERemoteService* UartService;
BLERemoteCharacteristic* Rx;
BLERemoteCharacteristic* Tx;

void scanCB(T_LE_CB_DATA* p_data) {
    foundDevice.parseScanInfo(p_data);
    if (foundDevice.hasName()) {
        if (foundDevice.getName() == String("AMEBA_BLE_DEV")) {
            Serial.print("Found Ameba BLE Device at address ");
            Serial.println(foundDevice.getAddr().str());
            targetDevice = foundDevice;
        }
    }
}

void notificationCB (BLERemoteCharacteristic* chr, uint8_t* data, uint16_t len) {
    char msg[len+1] = {0};
    memcpy(msg, data, len);
    Serial.print("Notification received for chr UUID: ");
    Serial.println(chr->getUUID().str());
    Serial.print("Received string: ");
    Serial.println(String(msg));
}

void setup() {
    Serial.begin(115200);

    BLE.init();
    BLE.setScanCallback(scanCB);
    BLE.beginCentral(1);

    BLE.configScan()->startScan(2000);
    BLE.configConnection()->connect(targetDevice, 2000);
    delay(2000);
    int8_t connID = BLE.configConnection()->getConnId(targetDevice);
    if (!BLE.connected(connID)) {
        Serial.println("BLE not connected");
    } else {
        BLE.configClient();
        client = BLE.addClient(connID);
        client->discoverServices();
        Serial.print("Discovering services of connected device");
        do {
            Serial.print(".");
            delay(1000);
        } while (!(client->discoveryDone()));
        Serial.println();

        UartService = client->getService(UART_SERVICE_UUID);
        if (UartService != nullptr) {
            Tx = UartService->getCharacteristic(CHARACTERISTIC_UUID_TX);
            if (Tx != nullptr) {
                Serial.println("TX characteristic found");
                Tx->setBufferLen(STRING_BUF_SIZE);
                Tx->setNotifyCallback(notificationCB);
                Tx->enableNotifyIndicate();
            }
            Rx = UartService->getCharacteristic(CHARACTERISTIC_UUID_RX);
            if (Rx != nullptr) {
                Serial.println("RX characteristic found");
                Rx->setBufferLen(STRING_BUF_SIZE);
            }
        }
    }
}

void loop() {
    if (Serial.available()) {
        Rx->writeString(Serial.readString());
    }
    delay(100);
}
```
#### Comment

- The BLEClient class is used to discover the services that exist on a connected BLE device. The discovery process will create BLERemoteService, BLERemoteCharacteristic and BLERemoteDescriptor objects corresponding to the services, characteristics and descriptors that exist on the connected device. These objects can then be used to read and write data to the connected device.

#

### 1.6 BLEUartService

```

#include "BLEDevice.h"

#define UART_SERVICE_UUID      "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHARACTERISTIC_UUID_RX "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHARACTERISTIC_UUID_TX "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"

#define STRING_BUF_SIZE 100

BLEService UartService(UART_SERVICE_UUID);
BLECharacteristic Rx(CHARACTERISTIC_UUID_RX);
BLECharacteristic Tx(CHARACTERISTIC_UUID_TX);
BLEAdvertData advdata;
BLEAdvertData scndata;
bool notify = false;

void readCB (BLECharacteristic* chr, uint8_t connID) {
    printf("Characteristic %s read by connection %d \n", chr->getUUID().str(), connID);
}

void writeCB (BLECharacteristic* chr, uint8_t connID) {
    printf("Characteristic %s write by connection %d :\n", chr->getUUID().str(), connID);
    if (chr->getDataLen() > 0) {
        Serial.print("Received string: ");
        Serial.print(chr->readString());
        Serial.println();
    }
}

void notifCB (BLECharacteristic* chr, uint8_t connID, uint16_t cccd) {
    if (cccd & GATT_CLIENT_CHAR_CONFIG_NOTIFY) {
        printf("Notifications enabled on Characteristic %s for connection %d \n", chr->getUUID().str(), connID);
        notify = true;
    } else {
        printf("Notifications disabled on Characteristic %s for connection %d \n", chr->getUUID().str(), connID);
        notify = false;
    }
}

void setup() {
    Serial.begin(115200);

    advdata.addFlags(GAP_ADTYPE_FLAGS_LIMITED | GAP_ADTYPE_FLAGS_BREDR_NOT_SUPPORTED);
    advdata.addCompleteName("AMEBA_BLE_DEV");
    scndata.addCompleteServices(BLEUUID(UART_SERVICE_UUID));

    Rx.setWriteProperty(true);
    Rx.setWritePermissions(GATT_PERM_WRITE);
    Rx.setWriteCallback(writeCB);
    Rx.setBufferLen(STRING_BUF_SIZE);
    Tx.setReadProperty(true);
    Tx.setReadPermissions(GATT_PERM_READ);
    Tx.setReadCallback(readCB);
    Tx.setNotifyProperty(true);
    Tx.setCCCDCallback(notifCB);
    Tx.setBufferLen(STRING_BUF_SIZE);

    UartService.addCharacteristic(Rx);
    UartService.addCharacteristic(Tx);

    BLE.init();
    BLE.configAdvert()->setAdvData(advdata);
    BLE.configAdvert()->setScanRspData(scndata);
    BLE.configServer(1);
    BLE.addService(UartService);

    BLE.beginPeripheral();
}

void loop() {
    if (Serial.available()) {
        Tx.writeString(Serial.readString());
        if (BLE.connected(0) && notify) {
            Tx.notify(0);
        }
    }
    delay(100);
}
```
#### Comment

- The BLECharacteristic class is used to create two characteristics, one for receive (Rx) and one for transmit (Tx), and added to a service created with the BLEService class.
- The required read/write/notify properties are set for each characteristic using the set__Property() methods, and callback functions are registered using the set__Callback() methods. The required buffer size is also set for each characteristic so that it has enough memory to store a complete string.
- When data is written to the receive characteristic, the registered callback function is called, which prints out the received data as a string to the serial monitor.
- When data is received on the serial port, it is copied into the transmit characteristic buffer, and the notify() method is used to inform the connected device of the new data.

#

# What are 4 roles of GAP for BLE

### 2.0 GAP Roles 

GAP (Generic Access Profil) is the layer that defines the topology of the BLE system. There are actually 4 roles defined by GAP: Central, Peripheral, Broadcaster and Observer.

- Central is in the “middle” of the topology. It is surrounded by Peripherals it can connect to, and it is the one that decides which Peripheral to connect to. In a way, it can be regarded as the active role in this topology.

- Peripheral is a device that usually takes the outside layers of a topology. Peripheral is meant to stay put until someone decides to connect with it. Example of a Peripheral is a temperature sensor located in a bus station. Anyone can come with a smartphone, see it while scanning for devices, then connect and get the data (the temperature value) from it.

- Broadcaster role does nothing more than transmitting data to its surroundings. It does so by constantly advertising, and usually has useful data in the advertising packet, data that is meant for everyone to see. Such a device does not require a receiver, as its only role is to broadcast to others, so it never accepts connections

- Observer The Observer is the opposite of the Broadcaster: it passively listens to BLE devices in its area and processes the data from the advertising packets it receives. It does not need a transmitter, as it sends nothing and is never meant to enter a connection.


![GAP Picture](https://electropeak.com/learn/wp-content/uploads/2019/10/ESP32-BLE-advertise-resized.jpg)
