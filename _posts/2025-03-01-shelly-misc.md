---
layout: post
title: 'Shelly Scripting - Misc'
categories: [Home Automation]
tags: [shelly]
---

This post captures a bunch of "miscellaneous" observations on the Shelly home automation devices.

## Device Info

Rhe synchronous `Shelly.getDeviceInfo()` method is [documented](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures/#shellygetdeviceinfo) as returning a `DeviceInfo` object. The [doc's `DeviceInfo` examples](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Shelly/#shellygetdeviceinfo-example) omit `name`, but as of 1.4.4, it's there:

```shell
% curl "http://${SHELLY}/rpc/Shelly.GetDeviceInfo" | jq .
{
  "name": "Main Switch",
  "id": "shellypro4pm-08f908f908f9",
  "mac": "08F908F908F9",
  "slot": 1,
  "model": "SPSW-104PE16EU",
  "gen": 2,
  "fw_id": "20241011-114451/1.4.4-g6d2a586",
  "ver": "1.4.4",
  "app": "Pro4PM",
  "auth_en": false,
  "auth_domain": null
}
```
