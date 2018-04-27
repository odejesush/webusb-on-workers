# WebUSB on Workers ##

## Introduction ##

[WebUSB](https://wicg.github.io/webusb/) is a Web API that allows web pages to
access connected USB devices with the user's permission. At the moment, WebUSB
is only available in the main thread and cannot be accessed by
[Web Workers](https://w3c.github.io/workers/). Additionally, a USB device
interface cannot distinguish between requests from multiple sources. Operating
systems therefore enforce the constraint that an interface can have only a
single owning user-space or kernel-space driver. Chrome, acting as a user-space
driver, similarly enforces that only a single JavaScript execution context can
claim an interface. As a result, sites which may have multiple windows/tabs must
coordinate on which one owns the device, and that is a problem for multi-page
Web apps that need to share access with all of its windows/tabs. Shared workers
provide an ideal context to hold that ownership because their lifetime is the
union of lifetimes of the tabs that they are connected to. Dedicated workers
would also provide the benefit of allowing heavy processing of data from USB
devices to be performed on the worker thread instead of the main thread.
Therefore, this proposal is to allow WebUSB to be accessed within Workers so
that pages can share device access and perform I/O operations on the device on a
separate thread.

## Goals ##

* Expose WebUSB API in DedicatedWorkerGlobalScope and SharedWorkerGlobalScope
  through the WorkerNavigator

## Non-goals ##

* New features to WebUSB are not being added

## Example Use Cases ##

An example use case for this feature would be a web-based IDE that can debug an
Android device through USB using a library such as webadb.js. Most IDE users
typically have multiple files open, so a web-based IDE can have more than one
browser tab open for different files. Therefore, the web-based IDE can use
WebUSB on a Shared Worker to allow all of the open tabs to use ADB on the
Android device for debugging while only requesting permission to access the
device once. Additionally, the Shared Worker will allow the pages to share a
state, and in this case, it can be used to synchronize log output from the
Android device on the pages.

Another example use case is a USB device that contains sensors to read
temperature, pressure, and humidity. The reading of the sensors on the USB
device can be performed on a Dedicated Worker so that the page can get readings
from the device in real time without blocking the page.

This API would also help to simplify some already existing projects, such as
using WebUSB to take a photo with a DSLR. This project currently uses a worker
to run a WebAssembly module that processes data from the DSLR to prevent the
main thread from being blocked. However, data from the DSLR can only by
retrieved on the main thread and must be sent to the worker through a
postMessage. This is an extra step that could be avoided with WebUSB being
accessible from a worker to begin with. Therefore, this feature would allow
projects that process data being read from or written to a USB device to be
simplified.

## Solution ##

The IDL for USB will include the `Exposed=(DedicatedWorker,SharedWorker,Window)`
extended attribute to specify the contexts in which the API should be accessible
from. The `requestDevice` method will also have the `Exposed=Window` extended
attribute so that the method is only accessible within the Window context.

The page has to request a device using `requestDevice` in order to receive
permission for the origin to use the device. Once permission has been granted,
the Web Worker can use `getDevices` to select a device from the list of devices
that the origin has access to. The Web Worker can then open the device and
perform I/O operations on it. On a `DedicatedWorker`, these operations can occur
asynchronously from the page, meaning that the page will not be slowed by heavy
I/O operations. On a `SharedWorker`, control of the device can be shared with
multiple pages from the same origin.

## Example ##

Using the example output from the Arduino device from
https://developers.google.com/web/updates/2016/03/access-usb-devices-on-the-web,
this example will demonstrate how to perform I/O operations and process the data
from the USB device from a `DedicatedWorker`. Data received from the USB device
can be sent to the document via postMessage and displayed on the document.

### index.html ###

```html
<!DOCTYPE HTML>
<html>
  <head>
    <title>WebUSB on Dedicated Worker Example</title>
    <script>
      let deviceWorker = new Worker('read-usb.js');

      // Request permission to access the device.
      function connectToDevice() {
        navigator.usb.requestDevice({ filters: [{ vendorId: 0x2341 }] })
            // Use the Vendor ID and Product ID to identify the USB device.
            .then(device => deviceWorker.postMessage({
              action: 'get-device',
              vendorId: device.vendorId,
              productId: device.productId,
            }));
      }

      // Read data from the connected device.
      function readDevice() {
        deviceWorker.postMessage({action: ‘read-device’});
      }
    </script>
  </head>
  <body>
    <button onclick=”connectToDevice()”>Connect Device</button>
    <button onclick=”readDevice()”>Read Device</button>
    <p>Device Output</p>
    <pre id="log"></pre>
    <script>
      // Update the pre element with output from the device.
      deviceWorker.onmessage = function(event) {
        let log = document.getElementById('log');
        log.textContext = event.data;
      };
    </script>
  </body>
</html>
```

### read-usb.js ###

```javascript
let selectedDevice;

onmessage = async function(event) {
  switch(event.data.action) {
    // Open the device specified with vendorId and productId.
    case 'get-device':
      let devices = await navigator.usb.getDevices();
      foreach (let device of devices) {
        if (device.vendorId === event.data.vendorId
            && device.productId === event.data.productId) {
          selectedDevice = device;
          selectedDevice.open();
          break;
        }
      }
      break;
    // Read data from the opened device and send to the page.
    case 'read-device':
      try {
        await selectedDevice.selectConfiguration(1);
        await selectedDevice.claimInterface(2);
        await selectedDevice.controlTransferOut({
          requestType: 'class',
          recipient: 'interface',
          request: 0x22,
          value: 0x01,
          index: 0x02
        });
        let result = await selectedDevice.transferIn(5, 64);
        let decoder = new TextDecoder();
        postMessage(decoder.decode(result.data));
      } catch(error) {
        postMessage(error);
      }
      break;
  }
}
```

### main.cc ###

```c++
// Third-party WebUSB Arduino library
#include <WebUSB.h>

WebUSB WebUSBSerial(1 /* https:// */, "webusb.github.io/arduino/demos");

#define Serial WebUSBSerial

void setup() {
  Serial.begin(9600);
  while (!Serial) {
    ; // Wait for serial port to connect.
  }
  Serial.write("WebUSB FTW!");
  Serial.flush();
}

void loop() {
  // Nothing here for now.
}
```

## Design Alternatives ##

The original idea was to allow only Shared Workers to use the WebUSB API.
However, allowing WebUSB to be used in a Dedicated Worker is also beneficial to
allow heavy I/O operations on a USB device to be offloaded to a worker thread.
Additionally, enabling the API to be used in a Dedicated Worker is as simple as
enabling the API to be used in a Shared Worker.

Another alternative design approach that was considered was to expose the entire
WebUSB API to the `DedicatedWorkerGlobalScope` and `SharedWorkerGlobalScope`.
This approach would require dealing with how the chooser prompt for selecting a
USB device to connect to should be displayed when used within a worker,
especially a Shared Worker which can contain multiple pages connected to it.
There would also be additional complexity in sending the request to show the
chooser from a worker thread to the main thread that contains the page.

The initial approach to allowing WebUSB to be used within the context of a
Dedicated Worker and Shared Worker was to create a base interface class called
`WindowOrWorkerUSB` that contained the attributes and methods of `USB` minus
`requestDevice()`. Then the existing `USB` class will contain only the
`requestDevice()` method and inherit from the newly created `WindowOrWorkerUSB`
interface. The `WorkerNavigator` interface would then use the
`WindowOrWorkerUSB` interface to expose the API to workers.

This idea was then changed to rename `USB` to `WindowUSB` and create a new
interface called `WorkerUSB` that inherits from `WindowOrWorkerUSB`. These new
interfaces would be added to the corresponding navigator interface. This
approach would have broken any existing implementations of WebUSB that use
`navigator.usb instanceof USB` to check if WebUSB is enabled. The whole idea of
creating a base `WindowOrWorkerUSB` interface was scrapped after learning that
extended attributes can be used to determine specific contexts in which an
interface, attribute, or method can be exposed in, therefore significantly
reducing the complexity of the implementation.

## Security ##

Allowing WebUSB to be used in the context of a Dedicated or Shared worker does
not come with any significant security considerations. Both the WebUSB and Web
Worker features already contain security measures such as preventing
cross-origin access and are also integrated into the Feature Policy, so
developers can control where and how these features are exposed in their web
applications. The Feature Policy is enforced across the pages and workers from
the same origin. In order to receive permission to access a USB device, the
`requestDevice()` method prompts the user to select a device to connect to.
Since `requestDevice()` will not be allowed to be accessed by workers, the
permission request should be done in the page. USB devices will only be opened
and claimed within one context at a time, and this is enforced by both the
WebUSB API and the operating system.
