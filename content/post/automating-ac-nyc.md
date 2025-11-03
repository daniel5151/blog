+++
title = "Turning a Dumb AC Unit Smart (Without Losing my Security Deposit)"
date = 2025-11-02
draft = true
tags = ["c++", "arduino", "homeassistant", "esp32"]
+++

**TL;DR:** You can do some really useful (albeit janky) stuff with just a servo motor, an esp32, and a dream.

* * *

My rental apartment's AC unit can only be controlled using these retro _analog_ knob-based controls, mounted right onto the unit. They work, but having to stand up and and fiddle with them all the time got annoying pretty quickly.

<p align="center">
  <img src="/blog/assets/automating-ac-nyc/aircon-controls-crop.jpg" width="400px">
</p>

Fortunately, there's an 'ol Prilik family saying that goes something like this: "remember kids, there's no problem in life that a servo motor and an esp32 can't fix"[^1]

And sure enough, after dropping ~$15 on parts, waiting for things to arrive from China, and spending a few hours iterating alongside my good friend Claude - I managed to hack together this beautiful mess:

<p align="center">
<img src="/blog/assets/automating-ac-nyc/v1-final-working-crop.jpg" width="400px">
</p>

What you're looking at here is a jerry-rigged **esp32-controlled servo motor**, coupled to one of my AC unit's knobs using a **shaft coupler**, all affixed to the AC's back-plane using a **cheap L-bracket** and a **binder clip** (with some industrial-grade **cardboard padding** for good measure).

This whole MacGyver'd up contraption talks to my **Home Assistant** instance over **MQTT**, which turns the AC unit on/off based on the state of a temperature sensor located in the same the room.

Et voila 🪄 ✨

I've managed to free myself from the schackles of having to get up off the couch just to tweak my AC!

> For all you professional embedded and mechatronics folks out there: I _strongly_ suggest you stop reading here.
>
> The rest of this blog post is a walkthrough of a _software_ engineer's approach to home automation and building custom hardware, and let me tell you: both the final product, _and_ the journey to get there, are _hella jank_.
>
> That said... if you're not afraid of a bit of jank: read onwards, and join me on this fun foray into how I managed to hack together some totally bespoke home-automation hardware with almost no budget, or experience!

<!--more-->

## Setting the stage

In April 2025, I (finally!) moved to New York City. After a brief apartment hunt, I managed to find a place I'm pretty happy with: the location is convenient, the building is fairly modern, and best of all - my rent is stabilized and _way_ below market value (woo!). There's not much to complain about!

...well, except for the AC situation.

See, for whatever reason, developers in NYC _really_ like using these loud, power-hungry, wall-mounted [PTAC units](https://en.wikipedia.org/wiki/Packaged_terminal_air_conditioner) in their apartment buildings. Some buildings might spend a bit extra in order to wire these unit up to wall-mounted thermostats... but often times, they'll just go with the cheapest option: completely _analog, unit-mounted knobs_.

Here's a picture of what I'm talking about:

<p align="center">
<img src="/blog/assets/automating-ac-nyc/aircon-controls.jpg" width="400px">
</p>

Yeah - guess which option _my_ building went with 🙃

## What are my options here?

Like any good renter, the first thing I did was gently ask my landlord if there was any way to "upgrade" the controls to something a bit more... modern. As expected, the response was roughly along the lines of "lol no, why would we do that?", which, to be fair, was basically the response I expected.

Well, no matter, like and good engineer - _surely_ I can hack my way out of this pickle?

After a bit of finessing, I managed to pop the cover off the unit, and expose its ~~soft underbelly~~ inner workings. Much to my surprise, not only did I find some info about the unit, but even a whole wiring diagram!

<p align="center">
<img src="/blog/assets/automating-ac-nyc/ptac-controls-and-wiring-diagram-crop.jpg" width="600px">
<img src="/blog/assets/automating-ac-nyc/ptac-serial-and-info.jpg" width="600px">
</p>

Now, I'm no expert when it comes to wiring diagrams, but by following the wiring, it certainly seems like this entire circuit is operating at _line voltage_, with nary a low-voltage digital signal I can hook into in sight.

Ok, maybe I can just splice in a [smart relay](https://www.amazon.com/smart-relay/s?k=smart+relay) somewhere? I'm no electrician, but it's probably not _that_ hard, right?

Well, maybe? But honestly, I didn't think this was gonna be a viable route for me.

Setting aside the fact that working with line voltage and HVAC equipment is a bit "spooky" for someone with zero electrical wiring experience, the bigger issue was that all the juicy wires I'd be interested in intercepting are stuffed _deep_ inside the AC unit. As far as I could tell, the only way to access those wires would be to yank the whole unit out of the wall... something that I wasn't particularly interested in doing. [This thread](https://www.doityourself.com/forum/air-conditioning-cooling-systems/577337-older-ice-cap-ptac-add-thermostat.html) from ~2017 reinforced my impressions that this would be far more trouble than its worth.

Ok, another idea: what if I just cut power to the unit using a [smart plug](https://www.amazon.com/s?k=smart+plug&crid=2WEKKKT1D6E9M&sprefix=smart+pl%2Caps%2C142&ref=nb_sb_noss_2)?

> Note: The internet was quick to warn me that toggling power to a running AC unit could potentially cause damage to the unit. I'm not an expert in these matters, but for the sake of science (and because I'm a bit stubborn), I nonetheless kept looking into this option.

Well, much to my chagrin, the unit plugs into the wall using one of those fancy NEMA 5-20P plugs, which is basically impossible to find a smart switch for!

<p align="center">
<img src="/blog/assets/automating-ac-nyc/NEMA_5-20P.svg.png" style="background-color: white; padding: 4px">
</p>

Well shoot!

If hooking into the wiring is a non-starter, and putting the unit behind a smart plug is a non-trivial... am I just out of luck?

Of course not!

It was time to seriously consider the "obvious" solution: why not just make a little robot to turn the dials for me?

## Dialing in the right approach

Looking at the unit, we find 2 dials:

1. **Mode Control:** A stiff, discrete dial, clicking between 6 "modes" of operation (Off, Lo-Cool, Hi-Cool[^2], Vent, Exhaust, and Heat)
2. **Temp Control:** A smooth, analog dial, connected to a simple bimetallic-strip based thermostat

This gave me two options to toggle the AC unit on and off:

**Option 1:** If I hook into the Mode Control dial, I'd need to:

- Leave the Temp Control dial set to "max cold"
- Buy a servo motor with enough torque to overcome the stiff action of the dial
  - ...which would probably need a 12V DC (if not more) power source, requiring extra circuity to power
  - ...and require some more robust mounting hardware, to counteract the torque, and ensure the servo stays in the right place
- Precisely calibrate the servo motor to rotate the dial the right number of degrees between the "Off" state and the "Cool" state

**Option 2:** If I hook into the Temp Control dial, I'd need to:

- Leave the Mode Control dial on "Lo-Cool"[^3]
- Buy a cheap, low-torque, low-power servo motor, _just_ powerful enough to rotate the fairly loose dial
  - ...that doesn't need a lot of mounting hardware to stay in the right place, given that the torque is fairly low
- Imprecisely yeet the servo motor all the way left/right, toggling the target temp between "really really hot" or "really really cold"

Based on how I just described these two options, guess which one I went with 🥰

> Option 2 certainly is the "jankier" of the two options, given that it relies on a second-order property (target temp) to power the unit on/off.
>
> But then again, I really wanted to sidestep a bunch of (slightly) tricky hardware engineering questions that my smooth-brained software-engineering self didn't want to figure out this time around.
>
> I think it was the right call!

## The road to V0

I was _fairly_ sure this was gonna work, but obviously, the only way to find our was to hack together a proof-of-concept (ideally - with the least number of new purchases as possible).

To cut a long story short - here's what I came up with for V0:

| Part                      | Cost                | Source                                         |
| ------------------------- | ------------------- | ---------------------------------------------- |
| ESP32 Dev Board           | $6                  | [Amazon](https://www.amazon.com/dp/B0DDPJQX3X) |
| Shaft Coupler             | $6.69               | [Amazon](https://www.amazon.com/dp/B0D4YBM6HB) |
| Servo motor + controllers | $2.66 ($8 / 3 pack) | [Amazon](https://www.amazon.com/dp/B0BG4ZCFLQ) |
| L Brackets                | free                | leftover ikea parts (from a LAIVA bookshelf)                            |
| screws                    | free                | leftover monitor parts                         |
| USB Cable + charger       | free                | found in the 'ol junk drawer                   |

**Total:** ~$16

And here's the result:

_(breadboard with the rest of the hardware out-of-frame)_

<p align="center">
  <img src="/blog/assets/automating-ac-nyc/v0-working-closeup.jpg" width="400px">
</p>

If you look closely, you'll notice that there's no actual mounting mechanism that connects the servo motor assembly to the PTAC - it's literally just "floating" in mid-air!

This is thanks to an ~~unintentional~~ ingenious part of this design: the L-brackets simply make the servo motor "bigger", such that when it rotates, it ends up "bumping" against the back wall, which resists the torque, and ensuring the torque is transfered down into the dial to rotate it.

But hardware is just one part of the story: how about the firmware?

> Sidenote: I'm leaving out a few intermediate steps that I took to get to this design:
>
> - I didn't get the right shaft-coupler the first time (or the second time (or the third time...)), so it took a few amazon returns until I found the right one.
> - Before buying the ESP32 Dev Board, I validated the servo motor + shaft coupler worked using a (really, really) old Arduino Leonardo I had lying around, and controlling it manually over serial (using a really long USB cable extending to my PC)
> - My first attempt at mounting this thing involved wooden skewers, a glue stick, and a cut up amazon box... a failed experiment, to say the least.

### Writing the Firmware

This firmware is _dead simple_.

I mean, think about it: all it needs to do is glue together 3 Arduino APIs: WiFi (for connectivity), MQTT (for pub/sub communication with Home Asisstant), and Servo (to rotate the dial).

And because this is a personal project, we can dispense with fancy features like OTA updates, dynamic configuration, nice UX, etc... Just hard-code creds/URLs/IPs in the firmware, manually re-flash it, and that's good enough!

_Obviously,_ this was something that I could write myself in a weekend or two of tinkering. Heck, I'd probably end up sidestepping the Arduino framework entirely, and end up writing a bare-metal firmware directly in Rust, just for kicks.

...but the NYC summer was scorching, and I was actively annoyed constantly babysitting the AC all the time, so I figured I might as well try this "vibe coding" thing that all the kids are doing nowadays.

And _hot damn!_ LLMs can _cook_ when it comes to writing one-off greenfield throwaway code!

> Ok, listen... before going any further - let me clarify that I'm not really a fan on AI coding assistants in general.
>
> I'm sure I could write a whole blog post about that topic alone... but if you're anything like me, the _last_ thing you want to read is another pro/anti-AI blog post. So don't worry - I'll keep my thoughts brief here:
>
> LLMs might be able to "code", but I've yet to see one adequately "Software Engineer". Using it to deliver large novel features in existing (non-AI) codebases is a sure-fire way to accumulate staggering amounts of architectural technical debt.

And with that disclaimer out of the way... let me gush about Claude.

...


Outline:

- building v0 hardware
  - asked claude to make esp32 host a web-ui for controlling the motor
  - first "huzzah" moment - holding the motor in one hand as I trigger esp32 motion from phone, and the knob rotated!
  - big question: how to mount this thing??
  - second "huzzah" moment - sitting on the couch, and using webui to turn knob back/forth
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



"Production" v1:

| Part                          | Cost                | Source                                                                                          |
| ----------------------------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| ESP32 Dev Board (but smaller) | $3.30               | [Temu](https://www.temu.com/goods.html?_bg_fs=1&goods_id=601100253124451)                       |
| Shaft Coupler                 | $2.42               | [Temu](https://www.temu.com/goods.html?_bg_fs=1&goods_id=601099519071638&sku_id=17592227065720) |
| Servo motor + controllers     | $2.66 ($8 / 3 pack) | [Amazon](https://www.amazon.com/dp/B0BG4ZCFLQ)                                                  |
| L Bracket                     | $0.50 ($4 / 8 pack) | [Amazon](https://www.amazon.com/dp/B0DBFX6HL6)                                                  |
| Nuts and bolts                | negligible[^4]      | [Amazon](https://www.amazon.com/dp/B09KS23KQ6)                                                  |
| USB cable + charger           | $5                  | Temu (take your pick)                                                                           |
| Binder Clip (bodge)           | negligible          | The Office 🤫                                                                                    |

**Total:** ~$14

[^1]: Translated from the original Russian, of course
[^2]: The only real diff between these two modes is how fast the dispersion fan runs. Empirically, hi-cool _does_ make the room cool a _bit_ faster... at the expense of the fan being extra-loud. I usually stick to lo-cool.

[^3]: This works thanks to a nice property my unit has: when the target temp has been hit, and no more cooling is needed - the unit goes totally silent and inert (until the temp goes back up, and the unit needs to kick back in)

[^4]: It was ~$7 for a pack that'll last me for years to come
