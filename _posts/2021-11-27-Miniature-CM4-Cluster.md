---
header:
  image: https://i.imgur.com/k63rKBx.jpg
  teaser: https://i.imgur.com/k63rKBx.jpg
toc: true
---

## Origin of the project
There are many different tiny computer cluster projects on the internet, but back in the day, this one from Michal Zalewski catches my eye. It is based on Intel Edison and uses its onboard WiFi to do a tiny cluster.
![](https://i.imgur.com/vM35KDH.png)  
Source: [https://lcamtuf.coredump.cx/edison_fuzz/](https://lcamtuf.coredump.cx/edison_fuzz/)

The closest thing we could build when CM4 isn't launched yet is this one from 8086.net
![](https://i.imgur.com/SLTOite.png)  
Source: [https://clusterhat.com/](https://clusterhat.com/)

From a slightly more functional standpoint, I want to create a cluster that at least has a gigabit ethernet interconnection. So that is why I hold off this project until Raspberry Pi Compute Module 4 (CM4)

After CM4 is announced, I think most people would agree that this is a great board for cluster projects. Here are my top three reasons:
1. Delicated Gigabit ethernet.
2. USB for remotely flashing the SD card.
3. A PCIe lane to attach a tiny NVMe drive for node storage.

So the project begins with the goal of utilizing the three key reasons above and making it small.

## System Architecture
![](https://i.imgur.com/t5aGn2S.png)

The system architecture diagram above is way too complex because there are essentially just four hubs.
1. Gigabit Ethernet Hub
2. USB Hub
3. GPIO/ADC "Hub" 
4. UART "Hub"

And a bunch of sensors for the CM4 temperature monitor and input power monitor plus a Fan controller.  

The temperature sensors are there even when the SoC has a temperature sensor built-in is because I want the fan controller to be able to operate independently from the cluster's software.

My original plan is to hook up the 5th ethernet port to an onboard USB 3.0 to gigabit ethernet and share the same USB 3 type-A connector with the USB 2.0 hub, so I only need to add a clean-looking USB3 interconnect board to connect the host to the cluster. But the chip I use is out of stock and I couldn't find a proper vertical USB 3 type-A connector, so I end up removing that.

The power architecture is based on the point-of-load(POL) concept, every node is fed with the input voltage and has its own DC-DC for powering 5V to CM4 and 3.3V to M.2 connector. Also, note that the 3.3V output from RPI4 is not enough to power the gigabit ethernet switch, so I still need another 3.3V DC-DC to power everything on the hat.

## Cluster Hat


### Gigabit ethernet hub

![](https://i.imgur.com/lrNJhoa.jpg)

Interestingly even though gigabit ethernet has been launched for more than 20 years, there are still few public datasheets & schemas for the switch. When I'm searching for possible candidates, I stood across this GitHub repo created by libc0607 [https://github.com/libc0607/Realtek_switch_hacking](https://github.com/libc0607/Realtek_switch_hacking) which contains the datasheet I need. I picked the RTL8397N since it is the only physically possible package to be placed on the top side of the board (I don't want to do a complex double-sided board). Note that Microchip does have ethernet switch ICs, but there are mostly larger QFP packages for the port count & speed I need.

The next issue is the interconnection between the PHYs without a transformer. Searching for the solution, I was kind of sad that most cluster board that contains PHY-PHY direct connection isn't open-sourced, like Turing Pi and Jetson Mate Cluster, but I'm guessing it is caused by the NDAs for the switch IC.

But after examine the boards' image and Ti's [Application Notes](https://www.ti.com/lit/an/snla088a/snla088a.pdf) "AN-1519 DP83848 PHYTER Transformerless Ethernet Operation". I went with the same design as Ti's notes, which is capacitor coupled and resistor terminated to 3.3V, but it turns out a simple .1uF capacitor coupling is enough to do the job. After stressing the gigabit ethernet switch I'm pretty confident that it works:
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Stressing the gigabit ethernet hub, interesting to see the activity led is actually sequentially blinked. <a href="https://t.co/f1turUlK2m">pic.twitter.com/f1turUlK2m</a></p>&mdash; will whang (@will_whang) <a href="https://twitter.com/will_whang/status/1441966251056631809?ref_src=twsrc%5Etfw">September 26, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Oh and ah...... Remember the original design has a pullup resistor to the 3.3V? I didn't design the cluster board to contain half of the termination resistors because I thought it is fine to use the hat's 3.3V on both sides, but it turns out CM4 really don't like that because CM4's 3.3V rail will power off when the enabled pin goes low, it will also power off the PHY chip and thus the pins got 3.3V without its VDD power, and it kills the PHY chip. I end up designing another project to utilize those CM4 with broken PHY, and here is a quick glance.
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">My implementation of Raspberry Pi Compute Module 4 Camera board:<br>1. On board Coral TPU (USB 2.0 &amp; OOS everywhere...)<br>2. HDMI out<br>3. USB host for onboard usb hub or device mode<br>4. Faster Wireless card: AX200 WiFi 6<br><br>I love CM4<a href="https://twitter.com/hashtag/RaspberryPi?src=hash&amp;ref_src=twsrc%5Etfw">#RaspberryPi</a> <a href="https://t.co/aWjYbvKFE3">pic.twitter.com/aWjYbvKFE3</a></p>&mdash; will whang (@will_whang) <a href="https://twitter.com/will_whang/status/1457543584400359425?ref_src=twsrc%5Etfw">November 8, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Also, you can see from the system architecture that I've attached the MII interface to the host, which should be able to turn this switch into a managed switch but I haven't got time to investigate it because I'm good with unmanaged switches for now.

### GPIO/ADC "Hub"
![](https://i.imgur.com/ziXHrWQ.jpg)

There are a couple of pins I'm interested in controlling on the CM4, namely USB boot to enable remote flashing, Enable to power cycle CM4, and RUN to reset SoC. (Side note: I was confused about the RST and RUN on the CM4, so that's why the RST pin is also hooked up, but it is just and status output and not used for this project). Moreover, I also want to monitor the power for each node. 

there are two choices: get a GPIO expander and an ADC or get an MCU. With the current chip shortages, prices, and board space, I went with RP2040, it got the GPIOs and four ADC pins matching the four daughter cards perfectly. 

It is definitely overkilled but I don't mind the additional 1~2 USD cost difference, but I do mind the software support and ease to use.
It takes about half an hour to set up the C SDK and start uploading to the chip via SWD plus another 15 minutes to modify the example to have a simple console to control the CM4s.

RP2040 hooks up to the host in multiple ways, an SWD for programming RP2040, USB and UART for communication, and I2C just for fun.

A little bit of side note here is that the current measurement originally comes from a current sense amplifier on the hat, but I added Ti's eFUSE chip onto the cluster board later and it has the current monitor output, so turn to that to save some board space.


### USB Hub

USB2517, easy to use, good datasheet. Thank you for the D+/D- pin swap options, it does significantly ease my layout effort.  
I was planning to use its pin strap options to configure the hub, but there doesn't seem to have the combination I need, so I end up using an I2C EEPROM to configure it. 


### RPI 4 UART hub
Another interesting spec for RPI 4 is that it supports up to 5 UARTs,(This is one of the reasons why my Atomic Clock project moves to CM4). Why not connect all of it to CM4?  
Again this is because I don't want to rely on the Cluster's USB device capability to emulate either ethernet gadget or serial gadget or both.

### Other misc stuff

Input power is also protected with TVS and eFUSE, with the 5 RPI 4's DC-DC converter and ethernet switch power on at the same time, it is great to have an eFuse to also handle the soft-start. Input power measurement is INA226 because INA219 is yet another popular OOS item. 

There is also a qwiic connector just right under the DC jack, which fits perfectly between the ethernet connector on RPI4 and the hat.

![](https://i.imgur.com/io2D1Gz.jpg)


The fan controller is EMC2301, the same one as the CM4 breakout board from the RPI foundation. It is kind of puzzling to see that there is no build-in kernel support on Linux even though this is a well-documented part and used by the RPI foundation... But hey, i2cset works and so does the os.system() works. Also, note that the 4-pin fan connector is carefully placed so that the pin won't short with the connector.


## CM4 daughter card
![](https://i.imgur.com/DtqzbrF.jpg)

The card is simple, breakout the pins for UART/GigabitEthernet/USB and the control pins, and add an M.2 connector for local NVMe storage. Camera connector because why not? I do have some space left there and it might be interesting to use it to hook up to multiple cameras, but in the future, I might move M.2 connector to the back of the board and remove it to save some space.

![](https://i.imgur.com/7g87k96.jpg)


Another not-so-common thing on this card is that most of the component is under CM4, including the DC-DC converter and SD card slot. Using a 3mm B2B connector, there is about 1.5mm height for the component under CM4, the SD card connector I pick is 503182-1852 from Molex, 1.45 mm - just enough to squeeze under the Z-height limit. I don't need the onboard WiFi so it is fine to just place it near the antenna keep-out zone.

![](https://i.imgur.com/TpqDevN.jpg)


Apart from the normal 5V DC-DC step-down for CM4, there is another pair of DC-DC for the M.2 connector, given that most M.2 NVMe device takes quite some power, it is better to have a dedicated DC-DC for it and not take the 3.3V from CM4 (600mA only).

Finally, because the board is still pretty empty, I've added a secure EEPROM for the clusters, but guess what is the ETA for ATECC608A? The stock is as empty as the board :D 

Input power also needs some additional parts because I want to do hot-plug, TVS1801 for over-voltage protection, TPS2420 for eFUSE, and soft-start.
![](https://i.imgur.com/ExjNV9M.jpg)



## Result
Now back to the entire project.

For the switch performance, I use iperf3 and use two of the nodes as servers and the other two as clients. The performance is as expected with the overhead on gigabit ethernet (~940MBits/sec)
![](https://i.imgur.com/y8Z9Fp6.jpg)

If I stress the ethernet switch, CPU (overclocked to 2000Mhz) on the cluster, and fan, the power peaked to about 32W.
![](https://i.imgur.com/gkWzm0Z.jpg)

The temperature stays at about 45 degrees C in a 22-degree indoor temperature with maximum fan speed, so I can tune down the fan speed to about 50% and maintain about 60 degrees.

Note that the temperature comes from the sensor on the cluster card, which is right under the SoC but thermally coupled with a thermal pad, so the temperature is slightly different from the SoC temperature report, which is about 5~8 degrees higher. Also, the temperature difference between the nodes can be quite significant.
![](https://i.imgur.com/VWGzP9A.png)

USB boot is easy, just flip the USB boot pin and use [rpiboot](https://github.com/raspberrypi/usbboot), and now I don't even need to take out the SD card if CM4 doesn't boot.
![](https://i.imgur.com/LuSsQgo.jpg)


## Reflection
No project is perfect, and here are some of my reflections:

### M.2 as interconnect for Cluster and Hat
M.2 connector is not manically reliable for supporting the CM4 card with heatsink, not to mention that the M.2 spec is 0.8mm for board thickness, so the cluster board is a little bit hard to remove the CM4 without bending the board too much because of it. To make things worse, the heatsink for CM4 I'm using is [this](https://www.waveshare.com/cm4-heatsink.htm) one from Waveshare. It is heavy enough that the CM4 board is slightly tilted & making detaching and re-attaching CM4 harder. This is why if you look closer to the CM4 card image, you can see I reverse mount the screw and standoff so that I can take it down before I remove the CM4 itself.

Looking back I should go with either mini-PCIe or PCIe x1 socket to increase the board thickness.

### Power sequencing 
Because RPI can not provide enough power for the 3.3V rail, I have a dedicated DC-DC for it, but it causes some issue for the USB Hub that it couldn't get the configuring data from EEPROM if I attached the USB cable before I powered up. 

### Point-of-load (POL) power supplies Vs Chip shortages
The reason why I'm using POL arch for the power tree is that it minimized the power drop to CM4 input power rail, but it costs five DC-DC converters to step down to 5V and an additional five for 3.3V rail. With the current silicon shortages, I could only buy just enough amount I need to prototype. Looking back I think going with two big chunky DC-DC for 3.3V and 5V might be a better way to navigate the current climate.
![](https://i.imgur.com/pSaWYqn.png)

## Next steps

The next steps are to finish a tiny NFS SSD based storage pool for the nodes and start using K3s to run graphana, influxDB, and some real-time AC data analysis from the [Atomic Nixie Clock](https://github.com/will127534/RaspberryPiAtomicNixieClock) (I over-estimate the processing power for what a single CM4 can do at the same time).  

I still hope that one day Google Coral TPU can work with RPI through the PCIe bus, so I can attach it to one of my nodes for a simple inferencing node.

## Final Notes & Thought

Raspberry Pi 4 compute module is a really neat product that it brings low-cost but software rich and still got quite some peripherals (although it could be better if the PCIe device support is better, also [please add IEEE 1588 support in kernel](https://github.com/raspberrypi/linux/issues/4151)). This might also be one of the reasons why CM4 has been essentially out-of-stock almost everywhere, it is such a deal.

All the PCB files (I'm using KiCad, at the time for designing the version I'm using is V5.99 but it finally just released its v6 RC), are released on Github, and both EEPROM image and RP2040 Code are also uploaded, feel free to grab it and work on your own projects!

[https://github.com/will127534/Miniature-CM4-Cluster](https://github.com/will127534/Miniature-CM4-Cluster)

{% include comments.html %}
