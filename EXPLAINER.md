# WebUSB on Workers ##

## Introduction ##

[WebUSB](https://wicg.github.io/webusb/) is a Web API that allows web pages to
access connected USB devices with the user's permission. At the moment, WebUSB
is only available in the main thread and cannot be accessed by
[Web Workers](https://w3c.github.io/workers/). This is a problem for
multi-window Web apps because they need to request access to the device for
each open page. Additionally, all of the I/O operations performed on the USB
device is done on the render process main thread. Therefore, this proposal is
to allow WebUSB to be accessed within Workers so that pages can share device
access and perform I/O operations on the device on a separate thread.

## Goals ##

* Expose WebUSB in WorkerGlobalScope
* Synchronize the state of a connected device in each context

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

## Solution ##

The IDL for USB will include the `Exposed=(DedicatedWorker,SharedWorker,Window)`
extended attribute to specify the contexts in which the API should be accessible
from. The `requestDevice` method will also have the `Exposed=Window` extended
attribute so that the method is only accessible within the Window context.

The page has to request a device using `requestDevice` in order to receive
permission for the origin to use the device. Once permission has been granted,
the Web Worker can use `getDevices` to select a device from the list of devices
that the origin has access to. The Web Worker can then open the device and
perform I/O operations on it. On a `DedicatedWorker, these operations can occur
asynchronously from the page, meaning that the page will not be slowed by heavy
I/O operations. On a `SharedWorker`, control of the device can be shared with
multiple pages from the same origin.

## Example ##

TODO(odejesush): Add an example.

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
