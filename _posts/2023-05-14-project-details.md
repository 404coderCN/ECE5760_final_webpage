---
layout: post
title: FPGA-based Power Estimator for DE1-SoC
featured-img: post_background
summary: Implementation details of our power estimator project, including hardware, software design and testing
mathjax: true
---

## Introduction

&nbsp;&nbsp;&nbsp;&nbsp;Power consumption is a critical consideration of a lot of technologies today. Phone producers, for example, always talk about their improvement in power management and how it makes their battery last longer. The balance between functionality and power is an interesting topic investigated by many. When we are doing labs, however, we rarely think about how much power we are consuming, because we are never limited by it. When we synthesized the giant drum in lab 3 for this class, we exhausted all DSP blocks on the FPGA, and we used a lot of M10K memory. This kind of extensive use of hardware is likely to result in high power consumption, and we are curious to find out what are some of the factors that could impact the power consumption of an FPGA. With this goal in mind, we built a circuit that measures the current flowing through the power-supply cable using I2C-based current sensor [MIKROE-2987](https://www.mikroe.com/hall-current-2-click) while assuming a stable 12V voltage supply, and implemented an I2C receiver with integrator in the FPGA that continuously increments the energy consumed. Based on obtained data, we built an intuitive user interface on VGA showing current, real-time power, and average energy to monitor the power consumption of DE1-SoC.

## Design & testing methods

### Design Overview

&nbsp;&nbsp;&nbsp;&nbsp;The overall design of our power estimator is shown in Figure 1. MIKOE-2987 uses hall effect to measure current through internally fused input pins and has extreme low series resistence around 1.2m $\Omega$ which will not effect current supplied to FPGA. This board uses I2C interface by SDA for data and SCL for clock to convey obatined current data. Based on specification, we built I2C master as receiver in FPGA to convert the bitstream to actual high-resolution current data and pass it to HPS with a valid signal. This master receiver is controlled by HPS to implement reset, start and stop. HPS will process the obatined data and display corresponding user interface on VGA. To accelerate debugging, we also implemented 7-segment displayer to display the measured current.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/lab4_overall_design.jpg" width="80" height="20">
<figcaption align="center"> Figure 1: Overall design of power estimator </figcaption>
</center>
</div>


### Hardware design

&nbsp;&nbsp;&nbsp;&nbsp;We all know the equation $P = I * V$ from physics class, and since we already know the power adapter of the FPGA delivers a constant 12V voltage, we need a way to get the current reading. With some research online, we decided to go with the MIKROE-2987 current measurement board. This device relies on hall effect to measure the current passing through the internally fused input pins, and it has an extremely low series resistance of 1.2m $\Omega$, which makes it an almost perfect ammeter. The output of this current sensor goes to a 12-bit ADC with an I2C interface. So, to read the data stored in this device, we have to establish I2C communication between it and the FPGA. We achieved this by developing a finite state machine that adheres to the I2C protocol specified by the [MCP3221 ADC](https://www.digikey.com/htmldatasheets/production/48559/0/0/1/mcp3221.html?utm_adgroup=Integrated%20Circuits&utm_source=google&utm_medium=cpc&utm_campaign=Dynamic%20Search_EN_Product&utm_term=&utm_content=Integrated%20Circuits&gclid=CjwKCAjwjYKjBhB5EiwAiFdSfp9ML477NVziVD5azNRU4RDcGAhePigK16BnE5FcO95hGj4eKhzXJRoCMEwQAvD_BwE) as shown in Figure 2. The I2C communication only involves 2 wires-- SDA for data, and SCL for clock. In our case, the SCL line is controlled by the master and the SDA line can be controlled by either the master or the slave depending on which state the system is in. According to the datasheet, each transaction between the master and the slave should be 8 bits long and is always followed by an ACK/NAK. To initiate a transaction, the master device has to signal the start condition by pulling down the SDA line from HIGH to LOW while SCL remains HIGH. Then, the master should send out 4-bit device bits followed by 3-bit address bits, default to 1001101, and a read/write bit where 1 represents READ and 0 is WRITE. It's worth noting that changes in the SDA line should always happen when SCL is LOW, remain stable at the HIGH portion of the clock (except start and stop condition), and there should be one clock pulse per bit of data. After the initial transaction of address, the slave device will respond with an ACK that signals it has received the data. To send an ACK, a device must pull the SDA line LOW at the HIGH portion of the 9th pulse of SCL. The slave device, in this case the MCP3221 ADC, can then start sending the data stored in its internal 12-bit register. Since each transaction is limited to 8-bit, each valid current reading takes two transactions with the first four bits always equal to 0000. An ACK from the master is also required between the two transactions. After a full data transaction, the slave can either continue to transmit without repeating the address bits, or the master can generate a NAK by pulling SDA HIGH when SCL is also HIGH, and end with a stop condition by pulling up SDA from LOW to HIGH when SCL is HIGH.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/i2c_protocol.png" width="80" height="100">
<figcaption align="center"> Figure 2: I2C protocol specified by MCP3221 </figcaption>
</center>
</div>
<br/>

&nbsp;&nbsp;&nbsp;&nbsp;Following the I2C protocol explained above, we developed an 11-state FSM in SystemVerilog which can be seen in Figure 3. In STATE_DATA_RECEIVED_UPPER and STATE_DATA_RECEIVED_LOWER, the master receives data from the slave and puts it in a shift register of 12-bits. At each negative clock edge in these two states, the respective counter will increment by 1 and state transition happens when the counter reaches 7. Because the SDA line can be controlled by both the master and the slave, we wrote a tri-state buffer to decide which device gets control of the line. The tri-state buffer comes with an enable signal that is HIGH when the master device acts as an input in the transaction, and SDA should become high impedance on the master side when this enable signal is asserted. Otherwise, SDA gets the output from master. The SCL clock line follows the input 200K clock signal whenever we're not sending start/stop condition, not waiting, or not in idle. A data valid signal is pulled HIGH when the master acknowledges the lower bits of data or when a NAK is asserted by the master. This signal will later be used to control our integrator in calculating accumulative energy.

&nbsp;&nbsp;&nbsp;&nbsp;From I2C receiver, we can obtain 12-bit data, however, we still need to convert it into actual current data which we set as 5.22 fixed-point data. The corresponding voltage can be computed as $dataout \over 2^{12}$ * 3.3V. According to specification, sensitivity of current sensor is 110mv/A and original point for 1.65V, therefore, we can compute the actual current as $I = $ ${V-1.65V} \over 100mV/A$.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/fsm_diagram.jpg" width="50" height="110">
<figcaption align="center"> Figure 3: FSM diagram of I2C interface </figcaption>
</center>
</div>

<br/>

&nbsp;&nbsp;&nbsp;&nbsp; The MCP3221 I2C interface also specified that the maximum clock speed it can achieve is 400kHz. This means the 50MHz FPGA clock will be too fast to run, so we added a clock divider to step it down to 200kHz. With the I2C interface figured out, we went on to implement an integrator that takes the current output, computes the unit energy consumption, and increments continuously by adding on the previous energy value. We added a stop signal to allow pausing of integration. Whenever stop is not asserted and data from the I2C interface is valid, we calculate the new unit energy by computing $V * I * dt$ and add it to the total energy consumed. The value of dt was determined by counting the number of pulses between two consecutive data valid signals using SignalTap. We counted 18 pulses, which matched the protocol (16 pulses for transmitting data and 2 pulses for ACK). This means our unit energy at every new sample of current is $V * I * {18} \over {200000}$. Despite the input current using a 5.22 fixed point value, the energy output of our integrator module was set to 9.23 fixed point, because energy is accumulative, and we need more bits in the integer part to prevent overflow. We also increment a counter every time the current becomes valid, and we will be able to determine the elapsed time since the FPGA starts running by computing (cycle_count - 1)*$18 \over 200000$. Here we disregard the time for the first transaction, i.e., we are not counting the first 27 cycles, because it includes time to transmit the address bits which is not representative of the transactions in between. Since our clock runs at a high frequency, these 27 cycles should be negligible, and we preserve enough precision for our time estimation.

&nbsp;&nbsp;&nbsp;&nbsp; Towards the end of the project, we noticed that the energy output from our integrator module is not precise enough, and the average power is always a bit lower than exepcetd. To addess this problem we extended the energy output from the integrator module to be 64 bits, adding more fractional precision to the value. However, the Qsys PIO has a maximum bitwidth of 32 bits, so we had to create two PIO ports, one to transmit the higher 32 bit and one to transmit the lower 32 bit. We then used a long long type variable to store the 64 bit value in HPS, and later converted it from fixed point to double by a division of $2^{23}$. Doubling the number of bits improved the precision of the reading.

&nbsp;&nbsp;&nbsp;&nbsp;To make debugging more intuitive, we implemented a simple segment decoder on FPGA to display measured current value. As mentioned above, current value is 9.23 fixed point, we used segment displayer 0 as sign, next two as integer and resting as fraction part. We built a mapping table using first 9-bit as integer and 4-bit among 23-bit after decimal point.


### Software design

&nbsp;&nbsp;&nbsp;&nbsp;With all the needed hardware instantiated, we started to add communication with the HPS. Because our goal is to have a graphical representation of data on a VGA display, it is more convenient to work on the software side. We took an incremental approach in designing our software to guarantee correctness. We first allow the flow of data from FPGA to HPS by adding PIOs on the Qsys bus. Each PIO is given an offset from the lightweight AXI master, and we were able to access the data by looking at address: offset + lightweight AXI master address. Since the output from our integrator module is energy instead of power, we need to do some conversions. The dynamic power at every given instance equals instantaneous current * 12V, and the average power equals accumulated energy/total time spent. Before actually plotting these data, we want to make sure our data make sense, so we print the value of current, dynamic power, average power, cycle count, and time on our serial monitor whenever the data valid signal is pulled HIGH. In this process, we realized that we need to intentionally put our program to sleep for some time, because the HPS is running at a much faster rate than the FPGA, so one HIGH portion of the data valid signal would correspond to hundreds of cycles for the HPS, thus generate a lot more data points than needed. After confirming the correctness of our data, we need a proper way to display it. We decided to display three graphs on the VGA: current vs. time, dynamic power vs. time, and average power vs. time. We limit the time scale on screen to 3 seconds, meaning the graphs would be erased and redrawn using new data every 3 seconds. The y-axis scale is determined by the variable. For current, it is limited by the capability of our current sensor, which can handle up to 2A. For dynamic power and average power, the maximum is $I_{Max} * V_{Max} = 2 * 12 = 24 W$. So, from the bottom of the y-axis, every addition of pixel in the y direction represents a $24 \over 119$ W increase in power. We also created a text box to the right of the screen to display the actual value of these variables. Our final VGA display interface can be seen in Figure 4.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/intro_pic.jpg" width="500" height="150">
<figcaption align="center"> Figure 4: Display of graphs on VGA </figcaption>
</center>
</div>

<br/>

&nbsp;&nbsp;&nbsp;&nbsp;After being able to plot the data on VGA, we start to transition more control to the HPS side. We first disabled the button control of the start, stop, and calibration signal, and adopted a pure software control. This requires a user interface that detects input from keyboard, and we were able to achieve it by using multiple pthreads. A user is able to choose a number between 1-3, and each corresponds to pressing reset, stop, or calibrate. When a stop is detected, the drawing will immediately stop and data display freezes, and a start resumes the drawing. Calibration, on the other hand, clears the energy and time data and starts them from 0 again. All graphs will clear and immediately start redrawing when calibration is selected. A demonstration of the effect of calibration is presented in the video below.

<video width="600" height="400" controls>
  <source src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/vid_demo.mp4" type="video/mp4">
  <source src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/vid_demo.ogg" type="video/ogg">
Your browser does not support the video tag.
</video>

<br/>

### Testing strategy

&nbsp;&nbsp;&nbsp;&nbsp;To ensure all modules in our design work as expected, we took on a hierarchical approach to testing. At the lowest level, we first tested our I2C state machine by writing a testbench that feeds in control signal, clock, and data, and verified the output waveforms in ModelSim. As seen in Figure 5, the master is able to send out the correct address bits, generate ACKs when appropriate, read the input data and pull the data valid signal high when a 12-bit transaction completes.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/modelsim_waveform.jpg" width="400" height="70">
<figcaption align="center"> Figure 5: Modelsim waveform of testbench for I2C master implementation </figcaption>
</center>
</div>

&nbsp;&nbsp;&nbsp;&nbsp;After the I2C interface proved to be working, we instantiated the modules in Quartus and verified signal feedback from the FPGA using SignalTap. SignalTap creates the real hardware circuit in the FPGA and gives us the actual output waveform generated by our circuit. Up until this time, we had been using a power supply to feed in current to the current sensor, so we know exactly what value to expect for the current output. We were also using buttons on the FPGA to control the reset, start and stop signal here, so we can easily trigger data capture for SignalTap. By comparing the current output on the power supply to the data output from SignalTap, we know whether our implementation is done correctly.

&nbsp;&nbsp;&nbsp;&nbsp; At the end of the project, we moved from using power supply to directly measuring current drawn by the FPGA. We placed a 2.1mm female DC power jack on a breadboard and connect to it the FPGA power adapter. The power pin of the power jack is connected to the P+ pin of the mikroe-2987 board using a jumper wire, current will pass through the board and reach its P- pin, where we connected another jumper wire to the positive end of a male jack to open end power cable. The male power jack is inserted to the FPGA, whereas it negative end is connected back to the ground pin of the female power jack on the breadboard. This setup forms a series circuit where the current measured by the current sensor should be the same as the current drawn by the FPGA. Figures 6 and 7 show how we set up this series circuit. We observe the graphs and different readings on screen to see if the data makes sense. We observed fairly constant reading of current and power in this setting, which is expected because the FPGA is always running the same program. We also switched to using pure software control instead of buttons at this stage, and we compare the behavior of the VGA output when asserting signals in software to what was expected. We were able to see stopping of graphs when stop is selected and resume of drawing at start. Calibration also works correctly as graphs were erased and time axes reset properly, and the data display of current and energy goes back to 0.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/post_background.jpg" width="250" height="50" >
<figcaption align="center"> Figure 6: Setup of circuit to directly measure current drawn by FPGA </figcaption>
</center>
</div>

<br/>

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/closer_look_ammeter.jpg" width="60" height="100" >
<figcaption align="center"> Figure 7: A closer look at circuit around current sensor </figcaption>
</center>
</div>

<br/> 

## Analysis
&nbsp;&nbsp;&nbsp;&nbsp; Now that we can estimate the power consumed by the FPGA, we want to find out what are the factors that can influence the power consumption. Since lab 3 implementation already used up most of the hardware on the FPGA, it was not a good place to start. We tried integrating our power estimator code with our lab 2 code but some of the configuration on Qsys cannot be set properly, which caused the bottom half of the VGA display to malfunction. At the end, we decided to investigate the power consumption difference between using the ethernet cable to communicate with the FPGA and using serial. We first run the power estimator with the ethernet cable plugged in. The average power in this scenario is around 7.72W as shown in Figure 8. We then disconnect the ethernet cable and calibrate the readings by asserting the signal from the serial interface. The average power immediately drops down to 6.24W as shown in Figure 9. We repeated this process several times to confirm our finding, and we were always able to see a near 1W drop in average power every time we disconnect the ethernet cable. We did some research online and discovered that the ethernet port does take up a good chunk of power in operation. We are, therefore, able to conclude the use of ethernet connection in working with the FPGA increases power consumption significantly. We also tried adding two thousand 32-bit counters in the FPGA, but the change in power consumption was 

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/with_ether_power.jpg" width="200" height="70" >
<figcaption align="center"> Figure 8: Readings with ethernet cable connected </figcaption>
</center>
</div>

<br/>

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/without_ether_power.jpg" width="200" height="70">
<figcaption align="center"> Figure 9: Readings with ethernet cable disconnected </figcaption>
</center>
</div>

## Future development
&nbsp;&nbsp;&nbsp;&nbsp;Unfortunately, with the limitation of time, we were not able to carry out more investigations, so we still have plenty of room for further testing and improvement. We can, for example, experiment with adding and removing the VGA and audio subsystem and observe how that affects power consumption. It would definitely be meaningful if we could come up with a way to integrate the power estimator with other labs so we can better understand their performance.

## Appendix A
The group approves this report for inclusion on the course website.
The group approves the video for inclusion on the course youtube channel.

## References
[**MCP3221 ADC Datasheet**](https://www.digikey.com/htmldatasheets/production/48559/0/0/1/mcp3221.html?utm_adgroup=Integrated%20Circuits&utm_source=google&utm_medium=cpc&utm_campaign=Dynamic%20Search_EN_Product&utm_term=&utm_content=Integrated%20Circuits&gclid=CjwKCAjwjYKjBhB5EiwAiFdSfp9ML477NVziVD5azNRU4RDcGAhePigK16BnE5FcO95hGj4eKhzXJRoCMEwQAvD_BwE)

[**DE1-SoC User Manual**](https://people.ece.cornell.edu/land/courses/ece5760/DE1_SOC/DE1-SoC_User_manualv.1.2.2_revE.pdf)

[**MIKROE-2987 Schematic**](https://download.mikroe.com/documents/add-on-boards/click/hall-current-2/hall-current-2-click-schematic-v100.pdf)

[**ACS711 Current Sensor Datasheet**](https://download.mikroe.com/documents/datasheets/ACS711-Datasheet.pdf)
