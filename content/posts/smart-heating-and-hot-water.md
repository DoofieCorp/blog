+++
categories = ["smart home"]
date = "2022-07-04T00:00:00+01:00"
description = "Walkthrough of my chosen smart heating solution"
keywords = ["Smart Home", "drayton", "wiser", "Smart Heating", "Home Assistant"]
title = "Smart heating and hot water"

+++

Smart heating systems enable you to control your home's heating and hot water from more than just your boiler programmer. They provide fantastic convenience and may even result in reducing your energy costs.

While there are a vast amount of solutions on the market for achieving this, I struggled to find one that suited all my wants:

 1. Must have a local interface for smart controls, not being able to turn on/off the heating or hot water due to an internet failure is not acceptable.
 2. Should have physical buttons for turning on/off heating or hot water, nobody should require a network connected device to control the home's heating.
 3. Allow for per-room heating controls.
 4. Be easy to retrofit.


## Contenders 

### Nest

<center>

![Photo of a nest thermostat wall mounted](/images/smart-heating-and-hot-water/nest.png)

</center>

Arguably one of the most popular and well-known solutions is the Nest thermostat. On Amazon.co.uk these units come in at a 4.5 star rating.

Requirement 1 is a huge deal for me, I don't want to be in a scenario where my mobile app for my heating stops working due to a [cloud outage](https://nymag.com/intelligencer/2019/06/google-cloud-outage-impacts-nest-smart-locks-and-a-c.html).

Nest sadly doesn't provide local APIs to interact with their units. While they do have a remote API as described by the [Home Assistant Documentation](https://www.home-assistant.io/integrations/nest/) there is a fee.

### Tado

<center>

![](/images/smart-heating-and-hot-water/tado.jpg)

</center>

Not a well-known household name, but it is a first page hit for "smart heating" on amazon.co.uk.

Sadly, this product fails to satisfy requirement 1.  However, on the upside, they do have a remote API that is free of charge, the [home assistant documentation covers](https://www.home-assistant.io/integrations/tado/). Additionally, they satisfy my want by providing TRVs to allow for heating control on a per room bases.

### Hive

<center>

![](/images/smart-heating-and-hot-water/hive.jpeg)

</center>

The hive units are a perfect example of why requirement 1 is important. in 2021 hive announced a [shutdown notice](https://www.hivehome.com/us/support) that is, if you had any of their smart heating devices they are no longer going to function. As a result of this action, I quickly discounted all Hive products and did no further research.

### Heatmiser

<center>

![](/images/smart-heating-and-hot-water/heatmiser.jpg)

</center>

The Heatmiser units are fantastic and satisfied all of my requirements.

Home assistant provides [native integration](https://www.home-assistant.io/integrations/heatmiser/) for Heatmiser DT/DT-E/PRT/PRT-E thermostats.

These units are a bit dated looking. Thankfully, the [heatmiser neo range](https://www.heatmiser.com/en/heatmiser-neo-overview/) also has a local API and there is a [3rd party home assistant integration via hacs](https://github.com/MindrustUK/Heatmiser-for-home-assistant).

The NeoAir range was a very close option for me. However, I could not determine a way to do controls on a per room level, they do not offer TRVs that integrate with this system. Only existing heating zones can be controlled.

In the event, I was to self build I would plan for Heatmiser early on.

### DIY with Shelly.Cloud

<center>

![](/images/smart-heating-and-hot-water/shelly.jpg)

</center>

[Shelly.Cloud](https://shelly.cloud/) provides [relays](https://shop.shelly.cloud/shelly-plus-1pm-wifi-smart-home-automation) which could replace your boiler programmer, [TRVs](https://shop.shelly.cloud/shelly-trv-wifi-smart-home-automation#597) which would allow for per room control, or standalone [thermostats](https://shop.shelly.cloud/shelly-h-t-wifi-smart-home-automation-1#160) to use your existing zones

While this approach could work and would satisfy all requirements it works out extremely DIY heavy and expensive.

My heating system has 3 zones (downstairs, upstairs and hot water). I would need a relay for each zone, each relay costs 15.90eur which results in a total of 50eur.

To may the relays look pretty and to add physical controls I would need a 3 gang switch. This can be estimated at 20eur.

I have 12 radiators, to provide per room controls I would need a TRV for each radiator. A four pack of TRVs cost 237.24eur which results in a total of 711.72eur.

This brings the total cost for this solution to little over 780eur and I would have to do a bunch of configuration to make it work (read: it would probably always be broken).

### Drayton Wiser

<center>

![](/images/smart-heating-and-hot-water/drayton.jpg)

</center>

A well known name in the UK heating system space and a first page hit on amazon.co.uk for "smart heating", the [Drayton Wiser System](https://wiser.draytoncontrols.co.uk/) by Schneider Electric provides the complete solution in my opinion.

They have solutions for [1](https://www.amazon.co.uk/Drayton-Wiser-Thermostat-Heating-Control/dp/B075GS4WFK?ref_=ast_sto_dp), [2](https://www.amazon.co.uk/Drayton-Wiser-Thermostat-Heating-Control/dp/B075GRPZQ2?ref_=ast_sto_dp) and [3](https://www.amazon.co.uk/Drayton-Wiser-Thermostat-Heating-Control/dp/B075GRPZQ2?ref_=ast_sto_dp) zone heating systems. Additionally, they offer solutions based around [wireless TRVs](https://www.amazon.co.uk/Drayton-Heating-Radiator-Thermostat-Amazon/dp/B08WCGYMQN/) rather than wireless thermostats to give per room control.

For my setup, I needed a 3 zone boiler programmer. Sadly this can only be bought in a [bundle](https://www.amazon.co.uk/Drayton-Wiser-Multi-Zone-Thermostat-Radiator/dp/B075GSQDZF) of two TRVs and a wireless thermostat, with each radiator covered with a TRV the wireless thermostat just becomes a waste. The bundle costs 244.80eur.

Achieving per room control with this system isn't cheap, but it is cheaper than the Shelly approach and it doesn't require manual configuration. Each TRV costs 46.31eur, as the boiler programmer bundle comes with 2 TRVs I only needed an additional 10 totalling a cost of 463.10eur.

The total cost of the system came to 707.9eur.

While the Drayton Wiser system doesn't have native home assistant integration, it does provide a local API and somebody has created a [3rd party integration](https://github.com/asantaga/wiserHomeAssistantPlatform). In addition to this, their mobile app automatically detects if it's on the same network as your boiler programmer, if it is it connects directly, if not it will connect via a Drayton relay.

The physical controls provided by this system are fantastic, at each TRV you can twist left or right to turn up or down the temperature in the room. From the boiler programmer, there are physical buttons to turn on/off hot water or the heating zones.

I have no regrets in about decision to go with the Drayton Wiser system. I don't believe any other systems on the market satisfy my wants as well as this system. I have had friends and family purchase this system from my recommendation, and all have been satisfied with it. Its app is not perfect and its window open detection could be improved but overall, I'm happy.
