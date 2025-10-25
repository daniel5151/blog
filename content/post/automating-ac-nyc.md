+++
title = "Turning a Dumb AC Unit Smart (Without Losing my Security Deposit)"
date = 2025-09-28
draft = true
tags = ["c++", "arduino", "homeassistant", "esp32"]
+++

If you're just interested in the TL;DR, I'm happy to oblige:

My rental apartment's AC unit can only be controlled using these retro _analog_ knob-based controls, mounted right onto the unit. They work, but having to stand up and and fiddle with them all the time got annoying pretty quickly.

<p align="center">
  <img src="/blog/assets/automating-ac-nyc/aircon-controls-crop.jpg" width="400px">
</p>

Fortunately, there's an 'ol Prilik family saying that goes something like this: "there's no problem in life that a servo motor and an esp32 can't fix"[^1]

And sure enough, for only ~$15, a few hours of trial and error, and some help from my good friend Claude - I hacked together this beautiful mess:

<p align="center">
<img src="/blog/assets/automating-ac-nyc/v1-final-working-crop.jpg" width="400px">
</p>

What you're looking at is a jerry-rigged esp32-controlled servo motor, coupled to one of my AC unit's knobs using a cheap L-bracket, all affixed to the AC's back-plane using a binder clip (and a bit of industrial-grade cardboard padding).

This MacGyver-esqe contraption talks to my Home Assistant instance, which in turn, remotely controls the AC unit based on the state of a temperature sensor located across the room.

Et voila! No need to stand up to tweak the AC anymore! How very magical 🪄✨

Now, for all you professional embedded and mechatronics folks out there: I strongly suggest you stop reading here. The rest of this blog post is a walkthrough of a _software_ engineer's approach to home automation and building custom hardware, and let me tell you: both the final product, _and_ the journey to get there, are _hella jank_.

That said, if you're not afraid of a bit of jank: read onwards, and join me on this fun foray into how I managed to hack together some totally bespoke home-automation hardware with almost no experience (or budget)!

<!--more-->

## How we got here

A few months ago, I (finally!) moved to New York City. After a brief apartment hunt, I managed to find a place I'm pretty happy with: the location is great, the building is reasonably modern, and the rent is stabilized (woo!). There's honestly not much to complain about!

...well, except for the AC situation.

See, for whatever reason, developers in NYC _really_ seem to like using these loud, power-hungry, wall-mounted [PTAC units](https://en.wikipedia.org/wiki/Packaged_terminal_air_conditioner) in their apartment buildings. Some buildings opt to spend a bit extra in order to wire these unit up to a wall-mounted thermostat... but other buildings instead choose to save a few bucks, and go with the cheapest option: using completely _analog, unit-mounted knobs_ to control the temperature and power mode.

Here's a picture of what I'm talking about:

<p align="center">
<img src="/blog/assets/automating-ac-nyc/aircon-controls.jpg" width="400px">
</p>

Yeah - guess which option _my_ building went with 🙃

## What are my options?

Like any good renter, the first thing I did was gently ask my landlord if there was any way to "upgrade" the controls to something a bit more... modern. As expected, the response was roughly along the lines of "lol no, why would we do that?", which, to be fair, was basically the response I expected.

Well, no matter, I'm an engineer - maybe I can build my way out of this pickle?

After a bit of finessing, I managed to pop the cover off the unit, and expose its ~~soft underbelly~~ inner workings. Much to my surprise, not only did I find some info about the unit, but even a whole wiring diagram!

<p align="center">
<img src="/blog/assets/automating-ac-nyc/ptac-controls-and-wiring-diagram-crop.jpg" width="600px">
<img src="/blog/assets/automating-ac-nyc/ptac-serial-and-info.jpg" width="600px">
</p>

Now, I'm no expert when it comes to wiring diagrams, but by following the wiring, it certainly seems like this entire circuit is operating at _line voltage_, with nary a digital signal I can hook into in sight.

Ok, maybe I can just splice in a [smart relay](https://www.amazon.com/smart-relay/s?k=smart+relay) somewhere? I'm no electrician, but it's probably not _that_ hard, right?

Well, maybe... but at this point, I was starting to get a bit worried going any further down this whole relay / re-wiring rabbit hole.

Setting aside the fact that working with line voltage and HVAC equipment is a bit "spooky" for someone with zero electrical wiring experience, the bigger issue was that all the juicy wires I'd be interested in intercepting are stuffed _deep_ inside the AC unit. As far as I could tell, the only way to access those wires would be to yank the whole unit out of the wall... something that I wasn't particularly interested in doing. [This thread](https://www.doityourself.com/forum/air-conditioning-cooling-systems/577337-older-ice-cap-ptac-add-thermostat.html) from ~2017 reinforced my impressions that this would be far more trouble than its worth.

Ok, another idea: what if I just cut power to the unit using a [smart plug](https://www.amazon.com/s?k=smart+plug&crid=2WEKKKT1D6E9M&sprefix=smart+pl%2Caps%2C142&ref=nb_sb_noss_2)?

> Note: The internet wisely warned me that cutting power to a running AC unit could potentially cause damage to the unit, depending on how the unit is wired, and how long the compressor has been running for. I'm not an expert in these matters, but for the sake of science (and because I'm a bit stubborn), I nonetheless kept looking into this option.

Well, much to my chagrin, the unit plugs into the wall using one of those fancy NEMA 5-20P plugs, which is basically impossible to find a smart switch for!

<p align="center">
<img src="/blog/assets/automating-ac-nyc/NEMA_5-20P.svg.png" style="background-color: white; padding: 4px">
</p>

Well shoot! What now?

## Playing with servos

Outline:

- Setting the stage
  - moved into new nyc apt
  - AC unit is ptac from 2007, barely any docs on it, controlled via two knobs (temp + "mode")
  - found thread: https://www.doityourself.com/forum/air-conditioning-cooling-systems/577337-older-ice-cap-ptac-add-thermostat.html
    - didn't want to void lease - I hear cutting power to/from unit isn't good
  - why not just have a servo turn the knob for me?
- Toe in the water - playing with servos
  - Leonardo https://docs.arduino.cc/hardware/leonardo/
  - hacked together some code to control servo using serial (thanks claude!)
- getting more confident, buying esp32 + shaft coupler
  - ...and then going through 3x more shaft couplers, before finding one that worked
- building v0 hardware
  - asked claude to make esp32 host a web-ui for controlling the motor
  - question: which knob to control?
    - two options:
      - option 1: set temp to max cool, and turn "mode" knob
      - option 2: set "mode" knob to always low-cool, and control temp
        - (only viable thanks to "fan cycle switch", which can be toggled to turn unit "totally off" if not cooling)
    - option 1 is a non-starter, as the servo I bought is too weak to rotate the mode knob (requires too much force)
      - ...yes, even at 12 volts, which I hackilly tested using a random wall adapter + some wires + electrical tape (recreate photo?)
    - option 2 it is!
  - first "huzzah" moment - holding the motor in one hand as I trigger esp32 motion from phone, and the knob rotated!
  - big question: how to mount this thing??
    - brain-blast: instead of trying to mount the servo in 3d space... just add something to make the servo "bigger", such that it bumps against wall when rotating?
    - hacked together using ikea brackets (from a LAIVA bookshelf) and random screws (from my monitor)
  - second "huzzah" moment - sitting on the couch, and using webui to turn knob back/forth
  - _list prices here_
- making it useful with home assistant
  - quick TL;DR on what is home assistant
  - step 1: getting home assistant to control this thing
    - decided to do things "right", and asked claude to write MQTT firmware
    - took a few iterations, but servo shows up as a "cover" that can be "opened" or "closed" (rotate all the way left/right)
  - step 2: knowing when to turn it on / off (i.e: thermostat)
    - ezpz! just use https://www.home-assistant.io/integrations/generic_thermostat/
    - I already have a smart temp sensor ([AirGradient ONE](https://www.airgradient.com/indoor/))
  - ...connect the two, and that's it - we're done!
- side-quest: making a second thermometer for my other room
  - digging around through old junk, I found an [Azure IoT Developer Kit](https://microsoft.github.io/azure-iot-developer-kit/docs/serial-communications/) (AKA: AZ3166)
    - from the time I did an internship on the Azure IoT team
  - of course, \~6 years later, and most cloud-based supporting infra is dead...
  - thankfully, not _too_ hard to sidestep all the cloud nonesense, and use as a programmable (arduino compatible!) microcontroller
    - actually getting the devkit working with the arduino IDE was a bit annoying https://github.com/microsoft/devkit-sdk
  - asked claude to write a firmware that let clients fetch a JSON blob with the current temp data
  - use RESTful Sensor integration in Home Assistant to make sensors from data (by polling device)
    - https://www.home-assistant.io/integrations/sensor.rest
  - end result: good enough! temp values are ~5 deg off... but for thermostat purposes - the trendlines are all that matter!
    - ...but then, I decided to throw the data into colab, and asked gemini to futz with the numbers, and it did a damn fine job!
    - easy to add a calibrated sensor to home assistant using a template-sensor: `{{ ((states('sensor.mxchip_temperature') | float) * 0.919 + 2.23) | round(2) }}`
- building v1 hardware
  - the _unfathomable luxury_ of having two rooms (living room + bedroom) also means that I have two aircons, and two rooms
  - i.e: I need to build another unit
  - i.e: I can't keep relying on one-off IKEA mounting brackets and monitor screws
  - went on a temu / amazon shopping spree for some parts
  - _list prices here_

Prototype v0:

| Part                      | Cost                | Source                                         |
| ------------------------- | ------------------- | ---------------------------------------------- |
| ESP32 Dev Board           | $6                  | [Amazon](https://www.amazon.com/dp/B0DDPJQX3X) |
| Shaft Coupler             | $6.69               | [Amazon](https://www.amazon.com/dp/B0D4YBM6HB) |
| Servo motor + controllers | $2.66 ($8 / 3 pack) | [Amazon](https://www.amazon.com/dp/B0BG4ZCFLQ) |
| L Brackets                | free                | leftover ikea parts                            |
| screws                    | free                | leftover monitor parts                         |
| USB Cable + charger       | free                | found in the 'ol junk drawer                   |

**Total:** ~$16

"Production" v1:

| Part                          | Cost                | Source                                                                                          |
| ----------------------------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| ESP32 Dev Board (but smaller) | $3.30               | [Temu](https://www.temu.com/goods.html?_bg_fs=1&goods_id=601100253124451)                       |
| Shaft Coupler                 | $2.42               | [Temu](https://www.temu.com/goods.html?_bg_fs=1&goods_id=601099519071638&sku_id=17592227065720) |
| Servo motor + controllers     | $2.66 ($8 / 3 pack) | [Amazon](https://www.amazon.com/dp/B0BG4ZCFLQ)                                                  |
| L Bracket                     | $0.50 ($4 / 8 pack) | [Amazon](https://www.amazon.com/dp/B0DBFX6HL6)                                                  |
| Nuts and bolts                | negligible[^3]      | [Amazon](https://www.amazon.com/dp/B09KS23KQ6)                                                  |
| USB cable + charger           | $5                  | Temu (take your pick)                                                                           |
| Binder Clip (bodge)           | negligible          | The Office 🤫                                                                                    |

**Total:** ~$14

[^1]: translated from the original Russian, of course

[^2]:

[^3]: It was ~$7 for a pack that'll last me for years to come
