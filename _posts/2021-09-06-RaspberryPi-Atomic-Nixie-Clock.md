---
header:
  image: https://i.imgur.com/K8xKzDb.jpg
  teaser: https://i.imgur.com/K8xKzDb.jpg
toc: true
---
![](https://i.imgur.com/vn76Owb.jpg)

## Sys Arch
![](https://i.imgur.com/zrFgwyR.jpg)

## Atomic Clock
### Introduction

The core of the project and the main reason I started this project is this miniature atomic clock I picked up from eBay a couple of years back. This is the atomic clock I can integrate into my nixie clock in a reasonable power (~5W) and size (51mm x 51mm) requirement. What's better is that it is designed as pin to pin replacement for existing OCXO, so it is easy to just use a standard OCXO for testing. It will also serve as my frequency standard in my Lab.   
![](https://i.imgur.com/AWIKEOg.jpg)

I collected two of them, but unfortunately, the first one doesn't reliably lock, so I tear it apart, and this is what the internal component looks like:  
![](https://i.imgur.com/hWFkc32.jpg)

You can see the VCO, PLL, ADC, DAC, and output DDS.  
![](https://i.imgur.com/R5MAKdu.jpg)

And this is the laser + physics package.  
![](https://i.imgur.com/ic7vWHZ.jpg)

Not surprisingly, this type of Rb clock (Coherent Population Trap 1) doesn't perform as well as other "standard Rb based Atomic clock," 3x10E-11 (1s), 1.6x10E-11 (10s), 8x10E-12 (100s). But I still have the bragging right to claim this is an atomic clock, and thankfully my second SA.31m is working just fine.  
![](https://i.imgur.com/nBTmPy5.jpg)

Although it should be an easy integration, heat dissipation is an issue. On my testing board, if there is no additional airflow or heatsink, this clock can go all the way to 60-degree range, which is far too close to the limit of 70 operation limit.
For the longevity sake and to make it cooler, I've got a CNC heatsink, and it is contacting the atomic clock through the cut-out underneath and Thermal Conductive Pads.  
![](https://i.imgur.com/g0FqmAZ.jpg)

I also placed a TMP117 between the board and MAC to ensure the temperature measurement is accurate. 
But to be sure the heat dissipation is enough, I leave a cut-out for a Fan.

Also, as the first image show, I have an aluminum enclosure; the goal for the outline is to get closer to the "[Divergence Meter](https://steins-gate.fandom.com/wiki/Divergence_Meter)" in the animate "Steins;Gate." 

### GPS discipline  
![](https://i.imgur.com/MylZTvy.jpg)

The GPS module providing the PPS is NEO-M8T, a specialized timing series but smaller than traditional LEA modules. I did look into the latest and greatest ZED-F9T, but I don't have the board space left nor my pocket.

I want to prevent DAC or ADC measurement temperature coefficient issues, so instead of doing the discipline in the analog domain, I used TDC7200 to measure the time directly. The interface is connected directly to iCE40 FPGA.
Time measuring in a high-level structure is similar to [TICC](https://www.febo.com/pages/TICC/). The PPS output is the Start, and FPGA will output Stop after 300,000 cycles.
The reading is then captured by FPGA and send back to RPI. After correcting the GPS PPS sawtooth error, the data is then sent to the calibration module.

Here is the resulting plot of the uncorrected and corrected PPS duration measurement.  
![](https://i.imgur.com/S7kvhkP.jpg)

The discipline algorithm currently is a simple error accumulator and sends the FEC adjustment digitally to MAC.

Here are some of the results:
ADEV, MDEV plot of the PPS. Calculated and plotted by Matlab script provided by M. A. Hopcroft 2 
![](https://i.imgur.com/lWnVdFT.jpg)
![](https://i.imgur.com/K7SvJug.jpg)

### Issues
The main issue I have right now is the discipline algorithm. Initially, I'm using a PID loop, but it turns out a simple error accumulator works better...  
![](https://i.imgur.com/I8tj5V4.jpg)
The second issue is that PTP support is not complete on Raspberry Pi 4 Compute Module. I have the PTP input connected to FPGA, but I am still can't use it as my PTP main clock.  
https://github.com/raspberrypi/linux/issues/4151

Thirdly, while researching discipline algorithm, [jackson-labs LNRB manual](http://www.jackson-labs.com/assets/uploads/main/Low_Noise_Rubidium_user_manual.pdf) indicated the lousy phase noise profile this atomic clock has. It is bad enough that they even placed an OCXO just to filter out the noise. 

Lastly, it's more of a conceptual question I have. GPSDO can correct frequency error, but how should I synchronize with the UTC accurately? i.e., how can I make sure the PPS output is aligned with UTC? Since every GPS PPS contains the noise, it seems like there is no way I can tell what it is the actual PPS.

## AC waveform monitor because why not

### Structure
The structure is simple, an AC transformer provides a low voltage AC waveform, and an ADC8681 16bit ADC is sampling it directly at a 50Khz rate. The data is then going through FPGA internal FIFO and processed by Raspberry Pi. The timing comes from the atomic clock to ensure the frequency accuracy of the mains voltage. Data is then stored in the 256G NVMe drive in the clock, and it will keep about 7 days recent waveform data. And just noted here that low capacity NVMe drive is actually quite reasonable priced given the performance and longevity, and I bet most computer enthusiast have some low capacity NVMe hanging around. And I love Raspberry Pi 4 compute module have the PCIe to utilize those.  

### Calibration
Calibration is done by comparing the AC mains Vrms measurement from Keysight 34410A and simple linear regression. The voltage does span about 6V during a day (interestingly, higher voltage when low power demand in the region and vice versa, maybe the resistance in the power line is causing this?)  
![](https://i.imgur.com/BISWYLH.jpg)
![](https://i.imgur.com/jqUseYu.jpg)

### Some interesting result
California doesn't have a reliable power grid, thanks to PG&E, so I did capture some power outage waveform.
![](https://i.imgur.com/TpGnuX1.jpg)
![](https://i.imgur.com/5QBKQeQ.jpg)

## Nixie Tube
This might be the least interesting part, so I'm just going to say that the brightness control is a PDM signal generated by the FPGA to maintain the display under low brightness cause I don't want to add additional light bulbs in the dark while I'm sleeping. Nixie tube has a minimum on-time requirement for the digit to turn on entirely, so typically, when people using PWM, it's either you accept the low PWM frequency or the brightness can't go down to a certain degree. But by using PDM instead of PWM, the brightness can scale down further.  
I still have the traditional PWM method on the top board as you can see in the Sys Arch (because it gives me individual Nixie Tube brightness control), so here is a global brightness control comparison using PDM and PWM. (Camera setting is the same.)  
![](https://i.imgur.com/COOWdHC.jpg)

The high voltage switch IC is HV5623 to simplify the control and board space, also it is controlled directly by the FPGA. To ensure the timing, FPGA will assert the loaded to registers in HV5623 once PPS raised and output the next "frame" to the chip immediately after.

## MISC
* The WiFi antenna is on the top board.
* Input power filtered by common mode choke and EMI filter. But I have a wrong sch for the eFuse.
* There is a programmable PLL (Si5351c) output just in case someday I want to replace the crystal on CM4. 
* FPGA Source Code and PCB files are on [Github](https://github.com/will127534/RaspberryPiAtomicNixieClock)
* Higher resolution internal images for the atomic clock are on [Flickr](https://flic.kr/s/aHsmW1FnTJ)  


## References
* 1 Symmetricom. A COMMERCIAL CPT RUBIDIUM CLOCK. (https://www-users.york.ac.uk/~jke1/Atomic_Clocks/Papers/Commercial%20CPT.pdf) Retrieved September 4, 2021.

* 2 M. A. Hopcroft (2021). allan (https://www.mathworks.com/matlabcentral/fileexchange/13246-allan), MATLAB Central File Exchange. Retrieved September 4, 2021.
