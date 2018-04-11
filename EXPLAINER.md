# WebUSB on Workers ##

## Introduction ##

WebUSB is a Web API that allows web pages to access connected USB devices with
the user?s permission. At the moment, WebUSB is only available in the main frame
and cannot be accessed by workers. This is a problem for multi-window Web apps
because they need to request access to the device for each open page.
Additionally, all of the I/O operations performed on the USB device is done on
the render process main thread. Therefore, this proposal is to allow WebUSB to
be accessed within Workers so that pages can share device access and perform I/O
operations on the device on a separate thread.

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

A new WorkerUSB object that contains all of the WebUSB API minus requestDevice
can be defined in the WorkerGlobalScope to allow for Web Workers to access
connected USB devices that the page origin has access to. The page has to
request a device using requestDevice in order to receive permission for the
origin to use the device. Once permission has been granted, the Web Worker can
use getDevices to select a device from the list of devices that the origin has
access to. The Web Worker can then open the device and perform I/O operations on
it. On a DedicatedWorker, these operations can occur asynchronously from the
page, meaning that the page will not be slowed by heavy I/O operations.

## Example ##

TODO(odejesush): Add example code or link to example.

## Design Alternatives ##

Instead of enabling WebUSB to be accessed from AbstractWorker, only allow the
API to be accessed from a SharedWorker. This approach would allow pages from the
same origin to share devices. However, exposing WebUSB to the
SharedWorkerGlobalScope could result in additional complexity, since
SharedWorkerGlobalScope inherits the WorkerNavigator object from AbstractWorker.
Additionally, we lose the benefit of allowing WebUSB operations to be performed
on a separate thread with DedicatedWorkers.

Another alternative design approach that was considered was to expose WebUSB API
including requestDevice to the WorkerGlobalScope. This approach would require
dealing with how the chooser prompt for selecting a USB device to connect to
should be displayed when used within a SharedWorker