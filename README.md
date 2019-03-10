# Octo-Redirect
Step-by-step instructions for setting up a Raspberry Pi 3B and a Raspberry Pi Zero W to remotely print.

## Overview
The following diagram roughly describes the connections and communications required.

| Raspi3B | ➡️ | ZeroW | with | rfc2217_server.py | ➡️ | Mega2560 |
|---|---|---|---|---|---|---|
| rfc2217://10.20.30.66:2217 | tcp | 10.20.30.66:2217 | rfc2217 port forwarding to serial | `/dev/ttyACM0` on microUSB with Type A OTG adapter cable | USB serial cable | Type B connector |

## Setup
Read the related [Setup.md](https://github.com/OutsourcedGuru/Octo-Redirect/blob/master/Setup.md) file for specific instructions.

|Description|Version|Author|Last Update|
|:---|:---|:---|:---|
|Octo-Redirect|v0.1.0|OutsourcedGuru|March 9, 2019|

|Donate||Cryptocurrency|
|:-----:|---|:--------:|
| ![eth-receive](https://user-images.githubusercontent.com/15971213/40564950-932d4d10-601f-11e8-90f0-459f8b32f01c.png) || ![btc-receive](https://user-images.githubusercontent.com/15971213/40564971-a2826002-601f-11e8-8d5e-eeb35ab53300.png) |
|Ethereum||Bitcoin|
