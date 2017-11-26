# i2c communications on ESP32

 By Official I2C protocol as [UM10204 I2C-Bus Specification and User Manual](https://Fwww.nxp.com/docs/en/user-guide/UM10204.pdf), A valid I2C transaction must be a `START`, at least one Data Byte, and a `STOP`.  The Arduino environment is base on the AVR processor series, These Atmel CPU's implemention the I2C protocol allows unlimited pauses between protocol elements.  The Espressif ESP32 implement enforces a Strict TimeOut between elements.  This has resulted in perceived I2C bus stability issues ([834](https://github.com/espressif/arduino-esp32/issues/834),[811](https://github.com/espressif/arduino-esp32/issues/811),[741](https://github.com/espressif/arduino-esp32/issues/741),[682](https://github.com/espressif/arduino-esp32/issues/682),[659](https://github.com/espressif/arduino-esp32/issues/659) and many more.  @lonerzzz created a partial solution [#751](https://github.com/espressif/arduino-esp32/pull/751).
 
  The I2C protocol arbitrates exclusive ownership of the bus to a single Master Device from a `START` signal to the next `STOP`.  A `ReSTART` operation is just a `START` signal inserted into this `START` -> `STOP` flow.  Schematically like this:
  
 `START` (Read or Write) -> `ReSTART` (Read or Write) -> `STOP`.
 
 The existing Arduino code base is reliant on the AVR's ability to infintely pause a i2c transaction.  The standard coding practice of:
```c++
// set internal address pointer in I2C EEPROM from which to read
Wire.beginTransmission(ID);
Wire.write(highByte(addr));
Wire.write(lowByte(addr));
uint8_t err =Wire.endTransmission(false); // don't send a STOP, just Pause I2C operations
if(err==0){ // successfully set internal address pointer
  err=Wire.requestFrom(addr,len);
  if(err==0){ // read failed
    Serial.print("Bad Stuff!! Read Failed\n");
    }
  else {// successful read
    while(Wire.avaiable()){
      Serial.print((char)Wire.read());
      }
    Serial.println();
    }
  }
```
May not function correctly with the ESP32, actually **usually** will not function correctly.  The current arduino-esp32 platform is built upon the [espressif-idf](https://github.com/espressif/esp-idf) which is built on FreeRTOS, a multi-process/processor operating system. The Arduino sketch is just one task executing on One of the Two processor cores. The TimeOut is triggered when the FreeRTOS Task Scheduler does a task switch between `Wire.endTransmission(false);` and the i2c activites inside `Wire.requestfrom()`. The maximum TimeOut interval is around 12ms. 

This rewrite of `Wire()` is designed to avoid this TimeOut from ever being possible.  To avoid the TimeOut this library uses a queuing system that does not allow an i2c transaction to start unless a `STOP` has been issued.  But, alas, this creates some incompatibilities with the pre-exisiting Arduino code base. The changes to the standard Arduino `Wire()` coding are minimal, but, they are necessary:
```c++
// new, maybe Excessive Error Return codes for @stickbreaker:arduino-esp32
typedef enum {
    I2C_ERROR_OK=0,
    I2C_ERROR_DEV,
    I2C_ERROR_ACK,
    I2C_ERROR_TIMEOUT,
    I2C_ERROR_BUS,
    I2C_ERROR_BUSY,
    I2C_ERROR_MEMORY,
    I2C_ERROR_CONTINUE,
    I2C_ERROR_NO_BEGIN
} i2c_err_t;

// set internal address pointer in I2C EEPROM from which to read
Wire.beginTransmission(ID);
Wire.write(highByte(addr));
Wire.write(lowByte(addr));
uint8_t err =Wire.endTransmission(false); 
// don't send a STOP, just Pause I2C operations

//err will be I2C_ERROR_CONTINUE (7), an indication that the preceding
//transaction has been Queued, NOT EXECUTED

if(err == 7){ // Prior Operation has been queued
// it will be executed when the next STOP is encountered.
// if sendStop had equaled true, then a successful endTransmission() would have
// returned 0. So, if this if() was if(err==0), the Queue operation would be
// considered an error.  This is the primary Difference.

  err=Wire.requestFrom(addr,len);
  if(Wire.lastError()!=0){ // complete/partial read failure
    Serial.printf("Bad Stuff!!\nRead of (%d) bytes read %d bytes\nFailed"
      " lastError=%d, text=%s\n", len, err, Wire.lastError(), 
      Wire.getErrorText(Wire.lastError()));
    }
  // some of the read may have executed
  while(Wire.avaiable()){
    Serial.print((char)Wire.read());
    }
  Serial.println();
  }
```

Additionally this implementation of `Wire()` includes methods to handle local buffer data blocks. These local buffer can be up to 64k-1 bytes in length.

### New Methods: 
```c++
    i2c_err_t writeTransaction(uint8_t address, uint8_t* buff, size_t size, bool sendStop);// big block handling
    size_t requestFrom(uint8_t address, uint8_t* buf, size_t size, bool sendStop);
    size_t transact(size_t readLen); // replacement for endTransmission(false),requestFrom(ID,readLen,true);
    size_t transact(uint8_t* readBuff, size_t readLen);// bigger Block read
    i2c_err_t lastError(); // Expose complete error
    void dumpInts(); // diagnostic dump for the last 64 different i2c Interrupts
    size_t getClock(); // current i2c Clock rate
    void setTimeOut(uint16_t timeOutMillis); // allows users to configure Gross Timeout
    uint16_t getTimeOut();
``` 

`transact()` coding is:
```c++
// set internal address pointer in I2C EEPROM from which to read
Wire.beginTransmission(ID);
Wire.write(highByte(addr));
Wire.write(lowByte(addr));

uint8_t err=Wire.transact(len); // transact() does both Wire.endTransmission(false); and Wire.requestFrom(ID,len,true);
if(Wire.lastError != 0){ // complete/partial read failure
  Serial.printf("Bad Stuff!! Read Failed lastError=%d\n",Wire.lastError());
  }
  // some of the read may have executed
while(Wire.avaiable()){
  Serial.print((char)Wire.read());
  }
Serial.println();

```

This **APLHA** release should be compiled with ESP32 Dev Module as its target, and
Set the "core debug level" to 'error'

There is MINIMAL to NO ERROR detection, BUS, BUSY.  because I have not encounter any of them!


Chuck.
