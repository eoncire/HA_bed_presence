# HA_bed_presence
HomeAssitant / ESPHome / NodeRed bed presence and automations


Using an ESP32 flashed with ESPHome, two thin film pressure resistors, some NodeRed magic on the backend and HomeAssistant to tie it all together.

Setup / Testing Video - https://youtu.be/Ei5JgMIUK8M

Demo Video - https://youtu.be/Ei5JgMIUK8M

## Overview

I recently remodeled my kitchen and added some pretty bright overhead lighting fixtures (Acenx Smart Dimmers) and motion control for them (Wyze Sense PIR).  A problem arose, our dog and cat will walk around at night and trigger the lights and even on a low brightness value based on time of day they still shine right into our bedroom.  I researched some options and this is what i came up with.  The thin film pressure sensors looked promising so i bought one and tried it.  After some testing for a few nights to get some values to work with and build automations on, this is what I've come up with.  I chose the ESP32 becuase it has several ADC (analoge to digital) pins and i wanted two sensors, one for each side of the bed.  The ESP8266 only has one ADC pin.

## Hardware

![ESP32](https://i.imgur.com/THQlqsE.png)

**ESP32** - I buy everything i can on Amazon becuase I'm impatient.  Again, I used an ESP32 versus an ESP8266 because they have several ADC pins.  The pressue sensor is analoge and needs an ADC pin to work.  I power the ESP32 via the onboard USB plug, I have a multi-port phone charger right next to my bed anyways.

**Thin Film Pressure Sensor** - There are several sizes and styles.  This is the first one i tried and it worked well.  It's 24" long, perfect to use two of them, one for each side of the bed.  https://www.amazon.com/gp/product/B07M6YYPQ3/ 

**5k Ohm Resistors** - I tried a couple different resistances, this was the best for my scenario.

**Breadboard and SOLID Jumpers** - I have a stockpile of these for projects.  I really, really like the solid preformed jumper wires.  Also extrememly helpful are some breadboard screw terminals.  All available on Amazon.

## Software

**ESPHome** - The first step was getting a value we can read in HA.  The actual code for ESPHome is very basic and could be more in depth.  I should probably use the built in functions to average the data but this works.  I'm using the ADC platform of the sensor core in ESPHome and reporting the value back to HA every second (again, overkill).  I'm assuming you know the ESPHome basics and not going into any more detial here.

**HomeAssistant** - Theres not a whole lot special in HA here, other than showing a readout for testing.  HA does control my lighitng which this sensor is part of, as well as a bedroom box fan on a smart switch.  The actual automations are all in NodeRed.

**NodeRed** - This is where the magic happens!  The main reason to do this project is to not allow the main kitchen lights to go above a certain brightness value if both sides of the bed are occupied.  I also have set up an automation to turn on the bedroom fan (smart switch) if EITHER side of the bed is occupied, then turn off after 5 minutes if NEITHER side of the bed is occupied.

## Automations

![NodeRed Flow](https://i.imgur.com/3ZXEO0q.png)

The basic flow is as follows.  A **state_changed** node for each pressure sensor.  That sends the value readout (voltage) of the ADC pin on the ESP32.  

Next is a switch node where we set our trigger values.  In my case anything above 0.50v means someone (not the dog or a child) is in the bed.  My switch node is set up as so. ![Switch Node Config](https://i.imgur.com/zDX1Oz3.png)

For some reason that i'm not quite sure of, when the sensor reads 0.0v is sometimes is reported as OFF to HA, so in the switch node I added a switch for that.  All values are strings FWIW.

One step ahead is the boolean_logic node.  BUT, before we get there we need to format our data being fed into it.  After the switch node in our flow is a **change** node.  That takes the lower level of our switch node (as well as the off state) and changes the payload to **0**.

Now we come to the **booleanlogicultimate** node.  https://www.npmjs.com/package/node-red-contrib-boolean-logic-ultimate  This is what really makes the automation work.  It takes two different topics and compares their payloads to be either true or false.  It has 3 different outputs (AND, OR, XOR) and can compare as many topics as you want.  What we're doing here is comparing the two pressure sensor values to be TRUE (above zero) or FALSE (equal to zero).  The setup for the boolean_logic node is as follows (standard).

![Boolean Logic](https://i.imgur.com/fEXIX4R.png)

I didn't totally understand how this node worked at first, but once it clicked it made perfect sense.  You need two different TOPICS to compare.  Using two different sensor entities will do that, each one is it's own topic (my understanding).  I'm leaving the values as strings which is fine for this node.  It'll convert stings to numbers, anything above zero will be treated as TRUE, a zero value will be treated as FALSE.  That's why I used the switch and change nodes after the sensor value to set our TRUE / FALSE limits for this logic node.

Now coming out of the boolean_logic node we have 3 outputs, AND, OR, XOR from top to bottom.  I'm using AND (both side of the bed occupied) and OR (either side of the bed occupied).

![Flow](https://i.imgur.com/X3AZcCk.png)

**AND** output is fed to a basic switch node to seperate TRUE and FALSE payloads so we can automate accordingly.  The boolean_logic node sets the message.payload to TRUE or FALSE based on the state.  I want to set a global variable to use elsewhere in NodeRed.  That is done with a **change** node set up as follows.

![Variable Setup](https://i.imgur.com/Um6YjFL.png)

I repeat the same setup for a FALSE payload, setting the global variable of mbr_occupied_and to FALSE, but with the help of a trigger node to "debounce" the variable.  For example, if the sensor value is disrupted for a second i don't want the value to change.  The **trigger** node is simple, but very helpful.  Both the TRUE and FALSE output from the switch node is fed into the trigger.  On receiving a FALSE payload it'll start a timer (5 sec), if it receives nothing in that time it'll pass the original message and set the global variable to FALSE.  If it sees another TRUE payload then it'll shut off and reset the timer.

![Trigger Node](https://i.imgur.com/8F5htNI.png)

Next is the bottom half of the boolean_logic output for the **OR** values.  If there's just one person in the bed there are still things automations i want done.

![Or](https://i.imgur.com/CHpuQI8.png)

Again, I set a global variable (mbr_occupied_or) to TRUE or FALSE with a switch node, and a trigger node to debounce it turning off.  I also added a **call_service** node in here to turn the master bedroom fan on when someone is in bed, then wait a couple minutes to turn off once the bed is empty.

Lastly is a **link_out** node on the two outputs of the boolean_logic nodes that I'm using.  This allows me to use the output of this flow elsewhere in NodeRed for other automations in case i don't want to use the global variable i set.

![Link Out](https://i.imgur.com/ttGqf7m.png)





