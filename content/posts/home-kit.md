+++
title = 'Home Kit'
date = 2024-04-26T22:18:09+08:00
draft = true
+++

HomeKit Accessory Protocol(HAP)‐Protocol used by the accessory to communicate with Apple devices. For example, an iOS device uses HAP to discover, explore, and interact with HomeKit accessories such as lights, door locks, and garage door openers.

HomeKit Accessory Protocol supports three transports:

- BluetoothLE

- TCP/IP(Wi‐Fi/Ethernet)

- UDP/IP(Thread)



Accessories are composed of services and characteristics. An example of an accessory is a ceiling fan with a light and a mister that sprays cool water.

Services group functionality in order to provide context. In the aforementioned example accessory, there are three services: a fan service to interact with the ceiling fan, a light service to interact with the light, and a mister service to interact with the spray mister.

A characteristic is a feature that represents data or an associated behavior of a service. The characteristic is defined by a universally unique type, and has additional properties that determine how the value of the characteristic can be accessed.

- The properties of perms and ev, for example, indicate read/write/notify permissions, and event notifications.

- The characteristic may also have other properties that further describe the characteristic. Properties like format describe the format of the characteristic’s value such as int or string, description contains a description string, and minValue, maxValue, and minStep refer to minimum, maximum, and step limits that also apply to the characteristic’s value.

- For example, the”10.4 Light Bulb”(page192) service contains a ”11.2Brightness”(page229) characteristic of type public.hap.characteristic.brightness with permissions of paired read (pr) and paired write (pw). The ”11.2 Brightness” (page 229) characteristic may have additional properties describing the value. For example, a format property can indicate that the value is an int number, and a unit property can indicate that the value is in percentage. Furthermore, other properties can be used to indicate a minimum value of 0, a maximum value of 100, and a step value of 1 via the minValue, maxValue, and minStep properties.

Profiles define a collection of services (and characteristics) and their behaviors that can be supported by an accessory to provide a consistent user‐experience for a feature.



A HAP accessory server is a device that supports HomeKit Accessory Protocol and exposes a collection of accessories to the HAP controller(s). 

A HAP accessory object represents a physical accessory on a HAP accessory server. 

A bridge is a special type of HAP accessory server that bridges HomeKit Accessory Protocol and different RF/transport protocols, such as Zigbee or Z‐Wave. A bridge must expose all the user‐addressable functionality supported by its connected bridged endpoints as HAP accessory objects to the HAP controllers.

For example, a bridge that bridges three lights would expose four HAP accessory objects: one HAP accessory object that represents the bridge itself that may include a firmware update service, and three additional HAP accessory objects that each contain a Light Bulb service.

Any accessory, regardless of transport, that enables physical access to the home, such as door locks, must not be bridged. Accessories that support IP transports, such as Wi‐Fi, must not be bridged. Accessories that support Blue‐ tooth LE that can be controlled, such as a light bulb, must not be bridged. Accessories that support Bluetooth LE that only provide data, such as a temperature sensor, and accessories that support other transports, such as a ZigBee light bulb or a proprietary RF sensor, may be bridged.



Instance IDs are numbers with a range of [1, 18446744073709551615] for IP accessories (see ”7.4.4.2 Instance IDs” (page 150) for BLE accessories). These numbers are used to uniquely identify HAP accessory objects within a HAP accessory server, or uniquely identify services, and characteristics within a HAP accessory object. The instance ID for each object must be unique for the lifetime of the server/client pairing.

Accessory instance IDs, aid, are assigned from the same number pool that is global across entire HAP accessory server. For example, if the first Accessory object has an instance ID of “1”, then no other Accessory object can have an instance ID of “1” within the Accessory Server.

Service and Characteristic instance IDs, iid, are assigned from the same number pool that is unique within each Accessory object. For example, if the first Service object has an instance ID of “1”, then no other Service or Characteristic objects can have an instance ID of “1” within the parent Accessory object. The Accessory Information service must have a service instance ID of 1.



- Softwaretokensmustbeprovisionedintheaccessorythroughfactoryprovisioningatthetimeofmanufactur‐ ing, or in‐field provisioning through a firmware update. In‐field provisioning may only be used for previously‐ manufactured accessories that have already been distributed or sold.

- The token must be decoded using base64 and provisioned in raw data format in theaccessory.

- The UUID generated by the Licensee server for each token provisioned must be registered with Apple Server as defined in HomeKit Software Authentication Server Specification before provisioning in the accessory in little endian format (16 bytes).



The setup code must conform to the format XXXXXXXX where each X is a 0‐9 digit. For example, 10148005. For the purposes of generating accessory SRP verifier (see ”5.7 Pair Setup” (page 55)), the setup code must be formatted as XXX‐XX‐XXX (including dashes). In this example, the format of setup code used by the SRP verifier must be 101‐48‐005.

setup code可以动态生成，或者固定

The following are examples of setup codes that must not be used due to their trivial, insecure nature: 

- 00000000

- 11111111
- 22222222
- 33333333
- 44444444
- 55555555
- 66666666
- 77777777
- 88888888
- 99999999
- 12345678
- 87654321

This identifier is an alphanumeric string of 4(0‐9, A‐Z) characters. This identifier is persistent across reboots but may change after factory reset of the accessory. This identifier must be different than the DeviceID, serial number, model, or accessory name and must be random for each accessory instance manufactured by an accessory manufacturer.

The ”5.4 Device ID” (page 52) should be converted to uppercase for the computation of the setup hash. SHA‐512 is used to generate the setup hash from a concatenation of the ”4.2.2 Setup ID” (page 43) followed by the ”5.4 Device ID” (page 52). The setup hash is the first 4 bytes of the SHA‐512 digest.

Examples: The SHA‐512 digest for an accessory with Setup ID ”7OSX” and a deviceID: E1:91:1A:70:85:AA is (”echo ‐n ”7OSXE1:91:1A:70:85:AA” | shasum ‐a 512”):

C9FE1BCFB89B86E218AB56C5F1986EE9B9CC954A06C3769AC6433D472793EA7189DE46F 2A0E043363D5E1E4401D066F17AAC6D9C9A993F439879A331CF55F6CB

The Setup Hash is:

C9FE1BCF
