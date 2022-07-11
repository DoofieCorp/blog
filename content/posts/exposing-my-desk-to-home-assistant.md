+++
categories = ["smart home", "tasmota"]
date = "2022-07-11T00:00:00+01:00"
description = "Walkthrough of how my desk connects to Home Assistant"
keywords = ["Home Assistant", "Alexa", "Desk", "Tasmota", "ESP"]
title = "Exposing my desk to Home Assistant"
+++

<cemter>

{{< youtube _GEJ1rDDnAo >}}

</cemter>

I have a sit/stand desk. It has two buttons for control, one goes up, the other goes down. It connects to a control box via an RJ45 plug.

I wanted to connect the desk to [Home Assistant](https://home-assistant.io) so that I could use Alexa to move the desk up and down, to go to specific height percentages, and to connect to automations e.g. Via the [Home Assistant OSX application](https://github.com/home-assistant/iOS) Home Assistant is made aware if my webcam is on/off, I can use a state change to on to automatically make the desk go up to encourage me to stand during meetings.

## Discovery

<center>

![Photo of the PCB of the controller](/images/exposing-my-desk-to-home-assistant/controller-pcb.png)

</center>

To understand how the controller worked I opened it. Its PCB was well labeled and I was quickly able to identify the cables for up/down actions. As shown in the below video, control of the desk was achieved by pulling the M1- and M1+ cables to ground.

<center>

{{< youtube gfmq5j_V7wA >}}

</center>

## Implementation

![ESP32](/images/exposing-my-desk-to-home-assistant/iot-box.png)

Using an ESP32 flashed with [Tasmota](https://tasmota.github.io/docs/) I could configure two GPIO pins to be relays and connected them to M1- and M1+.

Triggering these relays from Tasmota pulled the M1- and M1+ pins to ground allowing programmatic control of the desk.

To connect the ESP32 to the M1- and M1+ pins I put an RJ45 plug on one end of CAT6 cable and plugged it into the control box. On the other end, I just stripped back the cables and connected them to the ESP32. In addition to the M1- and M1+ pins I was able to power the ESP32 through the 5v and ground exposed by the control box. Lastly, to allow the original controller to work I joint another piece of CAT6 which was terminated with an RJ45 keystone so that the original control could plug in without any modification.

<center>

![Tasmota template](/images/exposing-my-desk-to-home-assistant/tasmota-template-config.png)

</center>

To be safe, I made use of some additional features in Tasmota. To ensure up and down options couldn't be used at the same time I configured an [interlock](https://tasmota.github.io/docs/Commands/#interlnck) and to make sure the relays were never accidentally left on I configured a [PulseTime](https://tasmota.github.io/docs/Commands/#pulsetime) to automatically revert them back to an off state after a set duration.

## Integration with Home Assistant

As it’s Tasmota based the ESP32 is automatically picked up by Home Assistant. Sadly Alexa doesn’t have any concept of a sit/stand desk, to work around this I exposed the desk as a blind using [cover_rf_time_based](https://github.com/nagyrobi/home-assistant-custom-components-cover-rf-time-based)

```
 - platform: cover_rf_time_based
    devices:
      office_desk:
        name: Office Desk
        travelling_time_up: 16
        travelling_time_down: 16
        close_script_entity_id: switch.office_desk_down
        stop_script_entity_id: script.office_desk_stop
        open_script_entity_id: switch.office_desk_up
        send_stop_at_ends: False
```