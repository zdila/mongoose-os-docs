# CCM (Cloud Communication Module)

CCM provides secure IoT connectivity for any microcontroller-based
product. Below is the summary of the provided functionality:

- Send MQTT messages to the cloud using MQTT or RESTful API
- Subscribe to MQTT messages from the cloud
- Device shadow / device twin (AWS, Azure, mDash)
- Provision CCM over BLE, WiFi access point, Serial (by the Host MCU)
- Update Host MCU over-the-air. Implemented for STM32 Hosts
- Update CCM itself over-the-air

## 1. Setting up hardware

- Install `mos` tool by following steps 1,2,3 at [mos quickstart](https://mongoose-os.com/docs/mongoose-os/quickstart/setup.md)
- Connect your device to your workstation and run the following command
  to flash the CCM firmware:
  ```
  mos flash https://mongoose-os.com/downloads/ccm/release/CCM-WROOM32.zip
  ```
- If you're using [Olimex Ethernet board](https://www.olimex.com/Products/IoT/ESP32/ESP32-EVB), please run:
  ```
  mos flash https://mongoose-os.com/downloads/ccm/release/CCM-ESP32EVB.zip
  ```
- Login to the [license manager](https://license.mongoose-os.com) and buy CCM licenses (one per device)
- Execute
  ```
  mos license
  ```

## 2. Setting up network
  
- Provision CCM to WiFi by executing
  ```
  mos wifi WIFI_NETWORK_NAME WIFI_PASSWORD
  ```
- Provision CCM to the cloud service:
   *  [mDash](/docs/mdash/intro.md): `mos config-set dash.enable=true dash.token=ACCESS_TOKE`
   * AWS IoT: `mos aws-iot-setup --aws-iot-region REGION`
   * IBM Watson: `mos watson-iot-setup`
   * Microsoft Azure: `mos azure-iot-setup --azure-hub-name YOUR_AZURE_HUB_NAME`
   * Google IoT: `mos gcp-iot-setup --gcp-project PROJECT_ID --gcp-region REGION --gcp-registry REGISTRY`
   * Private MQTT: `mos config-set mqtt.enable=true mqtt.server=HOST:PORT`

## 3. Wiring

| CCM pin   | Host MCU pin  | Notes  |
| -------   | ------------  | ------ |
| IO 32     | UART RX       | &nbsp; |
| IO 34     | UART TX       | &nbsp; |
| GND       | GND           | &nbsp; |
| VCC 5V    | VCC 5V        | Do not connect 3.3V |
| IO 14     | RST (reset)   | Optional. Host OTA |
| IO 12     | BOOT0         | Optional. Host OTA |

Note: this is the default pinout and it is configurable,
see configuration section at the end of this document.

Below is the example wiring for the CCM-EVAL and STM32 BluePill board:

![CCM module wiring example on STM32 BluePill](images/wiring-bluepill.png)



### 3.1 Olimex EVB board wiring

If you're using [Olimex Ethernet board](https://www.olimex.com/Products/IoT/ESP32/ESP32-EVB),
board, please use the following wiring:

| NUCLEO-F401RE             | ESP32-EVB         |
| ------------------------- | ----------------  |
| CN10.21 / USART1_TX, PA9  | UEXT1.4 (GPIO36)  |
| CN10.33 / USART1_RX, PA10 | UEXT1.5 (GPIO16)  |
| CN8.14  / nRST            | UEXT1.10 (GPIO17) |
| CN8.7   / BOOT0           | UEXT1.3 (GPIO4)   |
| CN8.5   / 3.3V            | UEXT1.1 (3.3V)    |
| CN8.8   / GND             | UEXT1.2 (GND)     |

Below is the example wiring for the CCM on Olimex EVB and STM32 Nucleo board:

![CCM module wiring example on STM32 Nucleo](images/wiring-olimex.png)


## 4. Preparing firmware .zip file for the Host MCU

Build a new firmware for your Host using your familiar development tools,
like IAR or Keil IDE, Arduino IDE, or whatever else. Assume that the resulting
firmware file is `FIRMWARE.HEX`. To be processed by CCM it needs to be converted to
a .zip firmware bundle. Here's how to do it:

```
mos create-fw-bundle -o fw.zip --name fw --platform ccm_host host_fw:type=STM32,src=FIRMWARE.HEX
```

## 5. Updating Host MCU over-the-air


During development stage, the easiest way to OTA the host controller is
to use `mos` command:

```
mos --port ws://IP_ADDRESS_OF_THE_CCM/rpc ota FIRMWARE.ZIP
```

In production environment, many other methods are available for triggering
the OTA:

- Periodically poll a given URL
- Use "upload new firmware" UI button mDash cloud service
- Use ota RESTful API on mDash cloud service
- Call OTA method on the CCM via MQTT
- Trigger OTA via the device shadow (AWS IoT, Azure, mDash)

### 6. Communication protocol

Host MCU communicates with CCM over the UART. The communication protocol
is JSON-RPCv2, where JSON-RPC frames are delimited by newline characters.
Per JSON-RPC standard, JSON frames with the `id` attribute require response,
whereas frames without the `id` attribute are notifications that do not
trigger any response.

For example, CCM sends a device shadow delta to the Host. This frame does
not have an `id` attribute, thus Host does not send a response:


```javascript
{"method": "Shadow.Delta", "params": {"on": true}}  // CCM -> Host
```

When Host receives a JSON-RPC frame, it must parse the frame and call
corresponding handler function. A handler function processes the request
and produces a reply, which is sent back to the CCM.
In the following communication example, a CCM calls a custom function `Sum` that
adds two numbers:

```javascript
{"id": 12007, "method": "Sum", "params": [2,3]}   // CCM -> Host
{"id": 12007, "result": 5}                        // Host -> CCM
```

And here is the example when CCM calls a non-existent function:

```javascript
{"id": 12008, "method": "This_Function_Does_Not_Exist", "params": true}   // CCM -> Host
{"id": 12008, "error": {"code": -32601, "message": "method not found"}}   // Host -> CCM
```

The communication is two-way. This is an example where Host
sends an MQTT message to the cloud:

```javascript
{"method": "MQTT.Pub", "params": {"topic": "t1", "message": "hello"}}   // Host -> CCM
```

### 7. mjson JSON-RPC library

For the Host, a compact C/C++ JSON-RPC library called `mjson` is available.
`mjson` is a single-header, open source, MIT-licensed library located at
https://github.com/cesanta/mjson. `mjson` makes it
easy to define RPC services on the Host MCU, and call RPC services on CCM.

The `msjon` usage pattern is as follows:

1. Initialise the library by calling `jsonrpc_init()`
2. Export any number of custom functions using `jsonrpc_export()`
3. In the event loop, read the UART, feed each read byte into `json_process_byte()`
4. At any time, you can call `jsonrpc_call()` to send a JSON-RPC frame to the CCM

That pattern works on any Host MCU, regardless the architecture and
development environment: on Nordic nRF, TI, PIC, Atmel, STM32, NXP, or
what have you.

It is easy to demonstrate it on the Arduino platform.
The Arduino sketch below shows how to create a `Sum` RPC service for adding
two numbers:

```c
#include "mjson.h"  // Sketch -> Add file -> add mjson.h

// Gets called by the RPC engine to send a reply frame
static int sender(const char *frame, int frame_len, void *privdata) {
  return Serial.write(frame, frame_len);
}

// RPC handler for "Sum". Expect an array of two integers in "params"
static void sum(struct jsonrpc_request *r) {
  int a = mjson_get_number(r->params, r->params_len, "$[0]", 0);
  int b = mjson_get_number(r->params, r->params_len, "$[1]", 0);
  jsonrpc_return_success(r, "%d", a + b);
}

void setup() {
  jsonrpc_init(sender, NULL, NULL,  "1.0"); // Initialise the library
  jsonrpc_export("Sum", sum, NULL);         // Export "Sum" function
  Serial.begin(115200);                     // Setup serial port
}

void loop() {
  if (Serial.available() > 0) jsonrpc_process_byte(Serial.read());
}
```

When this sketch is compiled and flashed on an Arduino
board, start Arduino Serial Monitor, type
`{"id": 1, "method": "Sum", "params": [2,3]}` and hit enter. You should
see an answer frame:

![CCM arduino sketch RPC example](images/rpc1.png)

### 8. RPC Services provided by the CCM

#### Shadow.Report

Update device's `reported` state on the cloud.
Parameters: arbitraty JSON value - a string, number, array,
or an object of an arbitrary complexity.

Example: update shadow reported state on the cloud: set "reported.on" to true

```c
jsonrpc_call("{\"method\":\"Shadow.Report\",\"params\":{%Q:%s}}", "on", "true");
```

Corresponsing JSON-RPC frame generated by the above call (note the absence of the `id`):

```javascript
{"method": "Shadow.Report", "params": {"on": true}}
```

#### MQTT.Pub

Send MQTT message to the cloud.
Parameters: a JSON object `{"topic": xx, "message": xx, "qos": xx}`.
`topic` and `message` must be strings, and `qos` must be a number 0 or 1.

Example - send "hello" message to the "t1" topic:

```c
jsonrpc_call("{\"method\":\"MQTT.Pub\",\"params\":{%Q:%Q,%Q:%Q,%Q:%d}}",
             "topic", "t1", "message", "hello", "qos", 1);
```

Corresponsing JSON-RPC frame generated by the above call (note the absence of the `id`):

```javascript
{"method": "MQTT.Pub", "params": {"topic": "t1", "message": "hello", "qos": 1}}
```

#### MQTT.Sub

Subscribe to the MQTT topic.
Parameters: a JSON object `{"topic": "..."}`. `topic` must be a string.
In order to receive MQTT messages from the cloud, Host must define
`MQTT.Message` method, which is going to receive
`{"topic": "...", "message": "..."}`. Example:

```c
static void mqtt_cb(struct jsonrpc_request *r) {
  char topic[100] = "", message[100] = ""; 
  mjson_get_string(r->params, r->params_len, "$.topic", topic, sizeof(topic));
  mjson_get_string(r->params, r->params_len, "$.message", message, sizeof(message));
  printf("Got %s -> %s\n", topic, message);
}

  ...
  jsonrpc_export("MQTT.Message", mqtt_cb, NULL);
  jsonrpc_call("{\"method\":\"MQTT.Sub\",\"params\":{\"topic\":%Q}}", "t/#");
```

### 9. RPC Services provided by the Host

#### RPC.List

Return a list of all RPC services exported by the Host.
Parameters: none.

NOTE: This is a built-in service.

Serial communication example:

```javascript
{"id": 1, "method": "RPC.List"}                             // CCM -> Host
{"id":1,"result":["Shadow.Delta","RPC.List","Sys.Info"]}    // Host -> CCM
```

#### Sys.Info

Return information about the Host.  Parameters: none.

NOTE: This is a built-in service.

Serial communication example:

```javascript
{"id":1,"method": "Sys.Info"}                             // CCM -> Host
{"id":1,"result":{"fw_version":"1.0", "arch":"unknown", "fw_id":"Nov 24 2018 09:10:27", "app":"posix_device"}} // Host -> CCM
```

#### Shadow.Delta

React on the device shadow change. This is a notification service, meaning
that the handler function must not send any return value. Instead, it could send
a `Shadow.Report` notification to the cloud, informing the cloud about the
changed state.

NOTE: This service should be implemented by the user
if user wants the device to be shadow-controlled.

Parameters: arbitraty JSON value - a string, number, array,
or an object of an arbitrary complexity.

Example serial communication (note the absence of the `id`):

```javascript
{"method": "Shadow.Delta", "params": {"on": true}}                  // CCM -> Host
{"method": "Shadow.Report", "params": {"on": true, "uptime": 132}}  // Host -> CCM
```

### 10. Provisioning CCM module

#### WiFi provisioning

##### WiFi provisioning via Access Point

When an unprovisioned CCM module boots up, it can open a WiFi Access Point.
It is possible to connect to the Access Point and use a RESTful service
to setup WiFi. An un-provisioned CCM is accessible at predefined IP
address `192.168.4.1`. Example WiFi setup:

1. Connect to a WiFi network `CCM-xxxxxx`
2. Run the following command (replace NAME and PASS with your WiFi name and password):

```
curl -d '{"config": {"wifi":{"sta":{"enable":true,"ssid":"NAME","pass":"PASS"}}}}' http://192.168.4.1/rpc
```

##### WiFi provisioning via Serial line

Using this method, a CCM module could be provisioned by the Host controller.
Call two RPC methods: `Config.Set` followed by the `Config.Save`:

Update CCM configuration - setup local WiFi network:

```c
jsonrpc_call("{\"method\":\"Config.Set\",\"params\":{\"config\":{\"wifi\":"
             "{\"sta\":{\"enable\":true,\"ssid\":%Q,\"pass\":%Q}}}}}",
             "NAME", "PASS");
jsonrpc_call("{\"method\":\"Config.Save\",\"params\":{\"reboot\":true}}");
```

Corresponsing JSON-RPC frame generated by the above call (note the absence of the `id`):

```javascript
{"id":1,"method":"Config.Set","params":{"config":{"wifi":{"ssid":{"enable":true,"ssid":"NAME","pass":"PASS"}}}}}
{"id":2,"method":"Config.Save","params":{"reboot":true}}
```

### 11. Factory reset

CCM provides a way to factory-reset the module. Factory reset clears WiFi
configuration, and opens an AP after boot for further WiFi provisioning.
Factory reset is triggered by the "long button press", i.e. setting the
factory reset GPIO pin low and holding it in the low state for some time,
defined by the timeout value.

See Configuration Reference section for the exact tunables.

### 12. Security

CCM modules provide high level of security, employing several layers
of protection:

- Flash encryption. Reading flash returns garbage. An attempt to copy
  the flash to clone CCM devices won't work.
- TLS 1.2 communication. Connections to the cloud services are secured
  by the modern TLS 1.2 standard.
- Ability to lock the configuration. Device vendor can choose the level
  of control end-user has over CCM - for example, completely disable any
  changes to configuration. By default, a CCM is provisioned to the cloud
  by vendor, and provisioned to WiFi by end-user.

### 13. Configuration reference

CCM configuration is a JSON document that is stored on CCM's flash and
could be managed by the following RPC functions:

- `Config.Get` - get configuration. Optional parameters: `{"key": "..."}`
  specifies the sub-tree to retrieve. If params is omitted, then the whole
  configuration tree is retrieved.
- `Config.Set` - set configuration. Parameters: `{"config": {...}}`
  specifies the sub-tree to change.
- `Config.Save` - saves configuration.

Below is a CCM configuration available for tuning:

```javascript
{
  "host": {
    "ota": {
      "enable": true,               // Enable/disable host OTA
      "chip_type": "STM32",         // Type of the host MCU chip. Currently only STM32 is supported.
      "stm32": {                    // STM32 Host OTA settings
        "boot0_pin": 12,            // Host BOOT0 pin
        "intf_type": "UART",        // Type of the host communication interface.
                                    // Currently only UART interface is supported.
        "rst_pin": 14,              // Host reset pin
        "uart": {                   // STM32 UART communication settings
          "baud_rate": 115200,      // UART baud rate used during update
          "rx_pin": 34,             // UART RX pin (connected to host TX)
          "tx_pin": 32              // UART TX pin (connected to host RX)
        }
      }
    },
    "rpc": {                        // Host RPC channel settings
      "enable": true,               // Enable RPC channel to host
      "intf_type": "UART",          // Type of the host communication interface.
                                    // Currently only UART interface is supported.
      "uart": {                     // RPC UART communication settings
        "baud_rate": 115200,        // UART baud rate used for RPC
        "rx_pin": 34,               // UART RX pin (connected to host TX)
        "tx_pin": 32,               // UART TX pin (connected to host RX)
        "fc_type": 0,               // Flow control: 0 - none, 1 - CTS/RTS, 2 - XON/XOFF
        "cts_pin": -1,              // UART CTS pin (connected to host RTS)
        "rts_pin": -1               // UART RTS pin (connected to host CTS)
      }
    }
  }
}
```

### Load Arduino sketch

Visit
https://github.com/cesanta/mjson/blob/master/example/shadow.ino
Arduino sketch example. It demonstrates how to implement LED on/off via
the device shadow. It works with any cloud that supports device shadow, for
example AWS IoT, Azure IoT, https://dash.mongoose-os.com.

See https://mongoose-os.com/ccm/ for video tutorial.
