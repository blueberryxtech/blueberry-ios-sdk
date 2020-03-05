# blueberryx-ios-sdk
Sample swift code for connecting to Blueberry

# Connecting and Getting Oxygenated Hemodynamic Response Data

Blueberry sends data across at either 100Hz or 50Hz sample rate, check the onboarding with device to determine what the setting is

To connect to blueberry use the following characteristics:
    //bluetooth
    public static let ACCESSORY_NAME = "blubry"
    public static let FNIRS_SERVICE = CBUUID.init(string:"0f0e0d0c-0b0a-0908-0706-050403020100")
    public static let FNIRS_WRITE = CBUUID.init(string:"1f1e1d1c-1b1a-1918-1716-151413121110")
    public static let FNIRS_READ = CBUUID.init(string:"3f3e3d3c-3b3a-3938-3736-353433323130")
    
To setup a connection to get raw data here is a sample snippet using corebooth:

func centralManager(_ central: CBCentralManager, didDisconnectPeripheral peripheral: CBPeripheral, error: Error?) {
        if peripheral == self.peripheral {
            print("Disconnected")
            self.headbandStatus.text = "disconnected"
            self.peripheral = nil
            
            // Start scanning again
            print("Central scanning for", BlueberryPeripheral.FNIRS_SERVICE);
            centralManager.scanForPeripherals(withServices: [BlueberryPeripheral.FNIRS_SERVICE],
                                              options: [CBCentralManagerScanOptionAllowDuplicatesKey : false])
        }
    }
    
    // Handles discovery event
    func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?) {
        if let services = peripheral.services {
            for service in services {
                if service.uuid == BlueberryPeripheral.FNIRS_SERVICE {
                    print("service found")
                    self.headbandStatus.text = "service found"
                    //Now kick off discovery of characteristics
                    peripheral.discoverCharacteristics([BlueberryPeripheral.FNIRS_WRITE,
                                                        BlueberryPeripheral.FNIRS_READ], for: service)
                    centralManager.stopScan()
                }
            }
        }
    }
    
    func peripheral(_ peripheral: CBPeripheral, didUpdateNotificationStateFor characteristic: CBCharacteristic,
                    error: Error?) {
        print("Enabling notify ", characteristic.uuid)
        
        if error != nil {
            print("Enable notify error")
        }
        
    }
    
    func peripheral(_ peripheral: CBPeripheral,
                    didUpdateValueFor characteristic: CBCharacteristic,
                    error: Error?) {
        
                //FNIRS DATA
 
                let bytes = characteristic.value;
                //hemodynamic response short path
                let arrayHBR : [UInt8] = [bytes![2], bytes![3], bytes![4], bytes![5]]
                var valueHBR : UInt32 = 0
                let dataHBR = NSData(bytes: arrayHBR, length: 4)
                dataHBR.getBytes(&valueHBR, length: 4)
                valueHBR = UInt32(bigEndian: valueHBR)
        
                //hemodynamic response long path
                let arrayHBO : [UInt8] = [bytes![6], bytes![7], bytes![8], bytes![9]]
                var valueHBO : UInt32 = 0
                let dataHBO = NSData(bytes: arrayHBO, length: 4)
                dataHBO.getBytes(&valueHBO, length: 4)
                valueHBO = UInt32(bigEndian: valueHBO)
        
                //print(valueHBO)

    }
    
     // Handling discovery of characteristics
    func peripheral(_ peripheral: CBPeripheral, didDiscoverCharacteristicsFor service: CBService, error: Error?) {
        if let characteristics = service.characteristics {
            //print(service.characteristics?.description)
            for characteristic in characteristics {
                
                if characteristic.uuid == BlueberryPeripheral.FNIRS_WRITE {
                    print("write characteristic found")
                    
                    // Set the characteristic
                    self.startChar = characteristic
                }
                
            }
            // Look at provided characteristics
            for characteristic in service.characteristics! {
                let thisCharacteristic = characteristic as CBCharacteristic
                
                // If this is the characteristic we want
                if thisCharacteristic.uuid == BlueberryPeripheral.FNIRS_READ {
                    // Start listening for updates
                    // Potentially show interface
                    self.peripheral.setNotifyValue(true, for: thisCharacteristic)
                    
                    // Debug
                    debugPrint("Set to notify: ", thisCharacteristic.uuid)
                    
                }
                
            }
        }
    }
    
    private func writeValueToChar( withCharacteristic characteristic: CBCharacteristic, withValue value: Data) {
        
        // Check if it has the write property
        if characteristic.properties.contains(.writeWithoutResponse) && peripheral != nil {
            peripheral.writeValue(value, for: characteristic, type: .withoutResponse)
        }
        
    }
    
    func centralManagerDidUpdateState(_ central: CBCentralManager){
        if central.state == .poweredOn {
            centralManager.scanForPeripherals(withServices: nil, options: [CBCentralManagerScanOptionAllowDuplicatesKey : true])
        }
    }
    
    func centralManager(_ central: CBCentralManager, didDiscover peripheral: CBPeripheral, advertisementData: [String : Any], rssi RSSI: NSNumber){
        
        var tempName = ""
        if (peripheral.name != nil){
            tempName = peripheral.name!
        }
        
        //print(tempName)
//        if (tempName != nil){
        if (tempName.lowercased().range(of:"blubry") != nil){
            self.peripheral = peripheral
            self.peripheral.delegate = self
            self.centralManager.connect(peripheral, options: nil)
           
            print("device connected")
            self.peripheral.discoverServices([BlueberryPeripheral.FNIRS_SERVICE])
            print("device service discovered")
        }
//        }
        
    }
    
    func peripheral(_ peripheral: CBPeripheral, didReadRSSI RSSI: NSNumber, error: Error?) {
        
    }
    
    func didReadPeripheral(_ peripheral: CBPeripheral, rssi: NSNumber){
        
        if (peripheral.name == "blubry"){
            self.peripheral = peripheral
            self.peripheral.delegate = self
            self.centralManager.connect(peripheral, options: nil)
            print("device connected")
            self.headbandStatus.text = "connected"
            self.peripheral.discoverServices([BlueberryPeripheral.FNIRS_SERVICE])
            print("device service discovered")
            self.headbandStatus.text = "service discovered"
        }
        
    }
    
    func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral){
        peripheral.readRSSI()
        peripheral.discoverServices([BlueberryPeripheral.FNIRS_SERVICE]);
    }
    
    
