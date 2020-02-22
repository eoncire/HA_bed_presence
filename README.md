# HA_bed_presence
HomeAssitant / ESPHome / NodeRed bed presence and automations


Using an ESP32 flashed with ESPHome, two thin film pressure resistors, some NodeRed magic on the backend and HomeAssistant to tie it all together.

Setup / Testing Video - https://youtu.be/Ei5JgMIUK8M

Demo Video - https://youtu.be/Ei5JgMIUK8M

## Overview

I recently remodeled my kitchen and added some pretty bright overhead lighting fixtures (Acenx Smart Dimmers) and motion control for them (Wyze Sense PIR).  A problem arose, our dog and cat will walk around at night and trigger the lights and even on a low brightness value based on time of day they still shine right into our bedroom.  I researched some options and this is what i came up with.  The thin film pressure sensors looked promising so i bought one and tried it.  After some testing for a few nights to get some values to work with and build automations on, this is what I've come up with.  I chose the ESP32 becuase it has several ADC (analoge to digital) pins and i wanted two sensors, one for each side of the bed.  The ESP8266 only has one ADC pin.

## Hardware / Installation

![ESP32](https://i.imgur.com/THQlqsE.png)

Overview
![1](https://i.imgur.com/uJFyjkN.png)


![2](https://i.imgur.com/FmgNfqn.png)
![3](https://i.imgur.com/w2Rp4wr.png)

The leads from the sensors are soldered to the wires.  I used two pieces of double sided foam tape to hold it together, the leads were really small and fragile, this should hold it.
![4](https://i.imgur.com/GXp5IBR.png)

I used a little piece of double sided foam tape at the ends to hold it in place.
![5](https://i.imgur.com/ohzYRhy.png)

**ESP32** - I buy everything i can on Amazon becuase I'm impatient.  Again, I used an ESP32 versus an ESP8266 because they have several ADC pins.  The pressue sensor is analoge and needs an ADC pin to work.  I power the ESP32 via the onboard USB plug, I have a multi-port phone charger right next to my bed anyways.

**Thin Film Pressure Sensor** - There are several sizes and styles.  This is the first one i tried and it worked well.  It's 24" long, perfect to use two of them, one for each side of the bed.  https://www.amazon.com/gp/product/B07M6YYPQ3/ 

**5k Ohm Resistors** - I tried a couple different resistances, this was the best for my scenario.

**Breadboard and SOLID Jumpers** - I have a stockpile of these for projects.  I really, really like the solid preformed jumper wires.  Also extrememly helpful are some breadboard screw terminals.  All available on Amazon.

## Software

**ESPHome** - The first step was getting a value we can read in HA.  The actual code for ESPHome is very basic and could be more in depth.  I should probably use the built in functions to average the data but this works.  I'm using the ADC platform of the sensor core in ESPHome to read the sensor value, then using the built in ESPHome automations and API, I'm turning on an input_boolean in HA directly from the ESP32.  My thoughts is that this approach takes another element out of the equation, the ESP is handling the logic to determine if the bed is occupied versus doing it in NodeRed.

There is two lines of this in my ESPHome code, one for each side of the bed and I just called them Pressure1 and Pressure2.

````
sensor:
  - platform: adc
    pin: GPIO35
    name: "Pressure 1"
    update_interval: 60s
    attenuation: 11db
    on_value_range:
     - above: 0.5
       then:
        - homeassistant.service:
           service: input_boolean.turn_on
           data:
            entity_id: input_boolean.pressure1
        - logger.log: Pressure1 On
     - below: 0.49999
       then:
        - homeassistant.service:
           service: input_boolean.turn_off
           data:
            entity_id: input_boolean.pressure1
        - logger.log: Pressure1 Off
````

**HomeAssistant** - Theres not a whole lot special in HA here, other than showing a readout for testing.  HA does control my lighitng which this sensor is part of, as well as a bedroom box fan on a smart switch.  The actual automations are all in NodeRed.

**NodeRed** - This is where the magic happens!  The main reason to do this project is to not allow the main kitchen lights to go above a certain brightness value if both sides of the bed are occupied.  I also have set up an automation to turn on the bedroom fan (smart switch) if EITHER side of the bed is occupied, then turn off after 5 minutes if NEITHER side of the bed is occupied.

## Automations

![NodeRed Flow](https://i.imgur.com/sWmsW2q.png)

Link to the flow - https://pastebin.com/2wp7h9nD

I want to know if the bed is in one of three states; empty, one person, two people (down the road I'll do conditional based on which side).  I'm doing this on two different fronts, one is a global variable in NodeRed (global.mbr_bed) and the other is an input_select in HA for a visual representation in the frontend.  The variable is either 0, 1, or 2 and can be used elsewhere in NodeRed pretty easily.  The input_select is either Empty, One, or Two.

The basic flow is as follows.  The start of the flow is an **events: state** node for each pressure input_boolean.  It reports either on or off.  I made a flow for each state i want to evaluate as true or false, and only one state is true at any given time.  The input_booleans go to a **node-red-contrib-bool-gate** (https://flows.nodered.org/node/node-red-contrib-bool-gate) which is the logic in the flow.  You tell that node specifically what data you want it to look at and evaluate, in this case it is the msg.payload of the two pressure sensor input_booleans.  You pick the gate type (AND, OR, XOR, NOR, etc) and it will report either true or false on the payload of msg.bool.  Here is the setup for that node for the flow that reads true when both pressure sensors / input_booleans are on;

![AND Gate](https://i.imgur.com/twYaTFG.png)

The AND gate will report true if both of the rules we created are true, pressure1 and pressure2 = on.  Again, the result of that node is msg.bool and it's a boolean, either true or false.  We can use our handy dandy switch node to split the result of this gate.

![Switch Node](https://i.imgur.com/YaphFIG.png)

Now we have something we can use for automations when both sides of the bed are occupied.  Next in the flow we set the global variable to "2" and set our input_select to "Two".

![Further Automating](https://i.imgur.com/zUoM26I.png)

Using a change node we change (set) our variable global.mbr_bed to "2".

![Change Node](https://i.imgur.com/Il5O3Wc.png)

Right after that we have a **call service** node to change the input_select to "Two".

![Call Service](https://i.imgur.com/fIqifJx.png)

That's the basics of the flow.  I made a different flow for each bed state, the only real difference is the gate type.  Only one state should be active at a time, so using this table i decided which type of gate for each state, AND for both sides of the bed, XOR for just one side of the bed, and NOR for neither.  Only one will evaluate as true at a time (important, otherwise your bedroom fan automation turns off in the middle of the night and you lose some WAF points).

![Gate Choices](https://i.imgur.com/XPFMCY4.png)

The trigger nodes aren't currently used, but I do use the **Link Out** node as another way to use this bed state in other automations like lighting.  You give that node a name, then use the Link In node elsewhere and select the name you want to link it to.

![Link Out](https://i.imgur.com/LHaxHj4.png)



## Lighting FLow Integration

Here is the disaster of my kitchen lighting flow....

![Kitchen](https://i.imgur.com/l2Qqyfh.png)

In my Kitchen tab i can drop the link in node and use that automation flow to trigger something, like turning off all of the lights in the house between night and sunrise as soon as both sides of the bed are occupied.

![Link In](https://i.imgur.com/Nwa6F6r.png)

Turn off all of the lights when between set time.

![Lights Out](https://i.imgur.com/G14kStM.png)

To make the lights react based on bed occupancy I feed the 3 Wyze motion sensors i have into a switch node.  

Actually, before the switch node is a simple **gate** node for another function.  I have an input_boolean set up in HA called Kitchen Motion.  That is exposed to my GoogleHome devices to be able to control via voice.  If that boolean / switch is turned on, it'll close the gate node and not allow any motion to be passed along the flow.  Say if we're sitting down to watch TV at night, I can tell google to turn off kitchen motion and the lights won't come on.  I have a small house that's very open, the kitchen / dining / living room all flow together and the kitchen lights are really bright.  

Anywho, the switch node is set up to check the status of the global variable which is set in the other flow when both sides of the bed are occupied.  If that variable is 1 then we go down one path, if 2 then another, and if 0 a third.  That's how I set the lights to turn on / off differently if the bed is occupied.

![Switch](https://i.imgur.com/SuUbVbd.png)

If the bed is occupied on both sides and there is motion detected, only the light above the sink (single LED bulb) and the dining room overhead light (3 small LED bulbs) are turned on, and only at 10% brightness.  The delay for off (trigger node FTW!) is quick, only 1 minute.

## Other Possibilities

Once we have the bed presence set up, other automations are a breeze!  Here's another example, fan control.  I have a box fan in the bedroom thats plugged into a smart switch.  I made another simple flow using the same setup of the two pressure booleans fed to a logic gate (OR gate in this example).  If either side, or both sides of the bed are occupied, it evaluates as true and will turn on the fan in the bedroom, then off once the bed is empty.

![Fan Control](https://i.imgur.com/Ez2CI37.png)


