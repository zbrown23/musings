# CWRU Motorsports systems design evolution

After shipping SR-25 and watching new leadership scope out the next generation of car, I wanted to provide some background of the design evolution for CWRUM's electrical architecture.

Firstly, semi-active and CANDAQ/CAN car evolved alongside each other, not because of each other. CANDAQ was initially conceptualized my freshman year of college before semi-active. Semi-active was merely the motivating force behind actually building a CANbus system.

Second, I may work in aerospace now, but I have always thought of the car architecture in upgrade "blocks", similar to military and aerospace nomenclature. 
These blocks are generally not backwards compatible with each other (exceptions occured) and require new hardware/software/engineering to make happen.

## Block 0
*Arduino Dog Clutch Controller*

This sucked. I had to repair it in the 2022 Arizona hotel room while the rest of the team got dinner. The UX sucked, the hardware was unreliable, the servo was dumb, etc etc.
### The Good
* it worked, sometimes.
### The Bad
* just about everything else

### More lore
* controlled the dog clutch on the car
* arduino nano + pololu reg + savox servo
* had to reupload the arduino sketch every time you needed to trim out the dog clutch setpoints
* lampcord going to a switch screwed into the steering wheel
* the DAQ was a seperate box (actually like three generations). At least two had ~~drinking~~ "party" games built in.
* **This architecture did not suck because the people who designed it were bad engineers. It sucked because they didn't have the resources to do something great. Huge ups to Mike, Arlo, Liam, Brian, Tom, Sam, Cole, Will, Amy, and everyone else for helping me and letting me try the crazy stuff below.**

## Block 1
*Servo Controller, Valve Control Unit (VCU), Steering Wheel*

Modus Operandi: **Build something that works**

### The Good
* Servo controller proved we can build boards that work well. It was a serious improvement over the existing solution in almost every way, with basically no downsides. Only big issue was in developing the big buck converter (bootstrapping power supply overcurrent due to mosfet Qgs that required reducing switching speed) and timeline vibin' due to integration concerns.
* really simple solution, integrated insanely fast. 
First VCU board was in hand in January, First steering wheel ~March, full integration in April/May. 
We closed out the second rev of VCU days before the first comp.
* Architecture lent itself well to inexperienced engineers. 
We didn't really know what we were doing, and there weren't a ton of "black box" things that could happen. 
The worst issues we had were around valve driver architecture and the harness that we had to build to integrate servo controller and semi-active on the wheel.

### The Bad
* Inflexible architecture. How do you add more stuff to a 400kHz I2C bus with a point-point extender?
* Harness was chonk. 12 pins on the wheel, 8 on VCU. This harness was also a point of failure and we didn't have semi-active for the 2023 summer. 
* A fault on one VCU channel could propogate to the rest of VCU. You could kill the whole car with one short. This is mega dumb.

### More lore
* Servo Controller was the first RP2040 board we did. 
Worked similar to the arduino solution but had fancy firmware that stored setpoints and could be trimmed with buttons, a more efficient regulator that I designed, and an LED to tell you what was actually going on. 
Also had provisions for future expansion... A CAN transciever. 
For the 2023 season, it just ran with a GPIO going to the steering wheel switch.
* Valve Control Unit was the first design of a semi-active controller. 
It used a Teensy Micromod + mosfets + a current shunt and amplifier to provide constant current control to the four shocks. 
Also featured CAN bus support. 
* Communication between the steering wheel and VCU was done with differential I2C, with multiple ICs on the steering wheel acting to provide driver input that VCU could actually act on.


## Block 2/2.1 < YOU ARE HERE >
*CANDAQ boards, distributed active suspension*

Modus Operandi: **Build something that works *well*.**

### The Good
* We built a "big boy" system with Cyphal/CAN.
* Adding and removing nodes is much, much easier now
* Everything talks on one network, so logging data is easy
* We can reuse major design chunks. See similarities between Honeybee and VCM, or Steering Wheel module and Elmeg, etc etc.
* As expandable as your heart desires. Only limited by CAN 2.0B bandwidth...
### The Bad
* Limited by CAN 2.0B bandwidth. 
No live camera streams on the car's bus, or high rate telemetry.
* System engineering and integration is more complex.
We need to design a system that can be laid out in the car reasonably well.
* Distributing the system means more enclosures, means more MechE work, points of failure, scope creep, problems.
* VCU on shock got run over during the endurance race in California. 
This was the first time my hardware had ever failed during a race, my junior year.
* Enclosures are still insanely goofy. Boxes are unnecessarily big or packaging them is unnecessarily difficult.
* Connector selection is just not great. This is not our fault and moreso stems from a lack of good, affordable options.
* **FIRMWARE SUCKS!!** Seriously, who let a bunch of EEs write firmware? Get some actual embedded software people involved and write well structured firmware. *The purple one* is leading this charge, I hope for nothing less than complete success on her part.

### More Lore
* there are two generations of this "block", mostly deliniated by the MCUs used and the integration approach. 
Gen 1 (SR-24) was super focused on sticking everything where it needed to go, and having only four signal nets on the car, 2x CAN and 2x power. 
This meant that the valve controllers were integrated into the shocks, all position sensing was to be done with Honeybee 3-axis mag encoders, and the whole show was going to be run by a crazy FPGA+Compute Module 4 board called *Arcadia*, after the Goose song.
In addition, Gen 1 focused on using the RP2040 wherever possible since it was an approachable and easy to use MCU. Unfortunately, PCBA packaging constraints, the lack of a real hardware CAN peripheral, and floating point performance axed the RP2040 for us and we moved to only STM32 in the Gen 2 system.
* Gen 2 wrapped up a lot of stuff into a couple boards. 
We had a front and rear controller (Hall Monitor Unit), that also integrated the valve controllers in its enclosure, as well as a much cleaner compute solution with Elmeg on a Raspberry Pi 5.
The main goal with Gen 2 was to consolidate hardware to lower costs and improve system integration, reducing overall vehicle complexity. This went reasonably well.

### Block 3
*THE FUTURE*

No idea what this entails. I have a couple of ideas though.
* Custom IMU/INS solution. We've tried this two other times, with LoCoBo and Gofastboatsmojito. Neither panned out particularly far, but this is something we seriously need.
* Higher bandwidth backbone. Switch the fore and aft of the car to something like an ethernet backbone, use Cyphal/UDP over it, then use Cyphal/CAN to communicate with end-nodes. This might require some "routing" software though which would be an interesting challenge.
* ECVT. The holy grail. This trade needs to happen with drivetrain involvement but there's no reason you can't spend time on it.
* E-clutch. A clutch pack that can get stuck anywhere in the driveline, within reason.
Could be used for e-4wd, torque vectoring, you name it.
* Better RF link. Block 2 uses wifi for everything. Find a longer range over air link and do something cool with it.
* X2, or a valve driver update. Redesign my goofy switch to be more efficient, do sensorless flow estimation, package it better.
* Home-built wheel force tranducer. You can do it.
* More driver augmentation. How can you use these sensors to let the driver know what's actually going on?
How can you help them make better decisions?