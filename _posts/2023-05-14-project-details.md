---
layout: post
title: FPGA based power estimator
featured-img: post_background
mathjax: true
---

## Introduction

&nbsp;&nbsp;&nbsp;&nbsp;Power consumption is a critical consideration of a lot of technologies today. Phone producers, for example, always talk about their improvement in power management and how it makes their battery last longer. The balance between functionality and power is an interesting topic investigated by many. When we are doing labs, however, we rarely think about how much power we are consuming, because we are never limited by it. When we synthesized the giant drum in lab 3 for this class, we exhausted all DSP blocks on the FPGA, and we used a lot of M10K memory. This kind of extensive use of hardware is likely to result in high power consumption, and we are curious to find out what are some of the factors that could impact the power consumption of an FPGA. With this goal in mind, we built a circuit that measures the current flowing through the power cable, and implemented an integrator in the FPGA that continuously increments the energy consumed. Using this information, we drew three graphs displaying current, dynamic energy, and average energy on a VGA display.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/intro_pic.jpg" width="500" height="150">
<figcaption align="center"> Figure 1: Display of graphs on VGA </figcaption>
</center>
</div>

## Design & testing methods

&nbsp;&nbsp;&nbsp;&nbsp;We all know the equation $P = I * V$ from physics class, and since we already know the power adapter of the FPGA delivers a constant 12V voltage, we need a way to get the current reading. With some research online, we decided to go with the MIKROE-2987 current measurement board. This device relies on hall effect to measure the current passing through the internally fused input pins, and it has an extremely low series resistance of 1.2m $\Omega$, which makes it an almost perfect ammeter. The output of this current sensor goes to a 12-bit ADC with an I2C interface. So, to read the data stored in this device, we have to establish I2C communication between it and the FPGA. We achieved this by developing a finite state machine that adheres to the I2C protocol specified by the [MCP3221 ADC](https://www.digikey.com/htmldatasheets/production/48559/0/0/1/mcp3221.html?utm_adgroup=Integrated%20Circuits&utm_source=google&utm_medium=cpc&utm_campaign=Dynamic%20Search_EN_Product&utm_term=&utm_content=Integrated%20Circuits&gclid=CjwKCAjwjYKjBhB5EiwAiFdSfp9ML477NVziVD5azNRU4RDcGAhePigK16BnE5FcO95hGj4eKhzXJRoCMEwQAvD_BwE). The I2C communication only involves 2 wires-- SDA for data, and SCL for clock. In our case, the SCL line is controlled by the master and the SDA line can be controlled by either the master or the slave depending on which state the system is in. According to the datasheet, each transaction between the master and the slave should be 8 bits long and is always followed by an ACK/NAK. To initiate a transaction, the master device has to signal the start condition by pulling down the SDA line from HIGH to LOW while SCL remains HIGH. Then, the master should send out 4-bit device bits followed by 3-bit address bits, default to 1001101, and a read/write bit where 1 represents READ and 0 is WRITE. It's worth noting that changes in the SDA line should always happen when SCL is LOW, remain stable at the HIGH portion of the clock (except start and stop condition), and there should be one clock pulse per bit of data. After the initial transaction of address, the slave device will respond with an ACK that signals it has received the data. To send an ACK, a device must pull the SDA line LOW at the HIGH portion of the 9th pulse of SCL. The slave device, in this case the MCP3221 ADC, can then start sending the data stored in its internal 12-bit register. Since each transaction is limited to 8-bit, each valid current reading takes two transactions with the first four bits always equal to 0000. An ACK from the master is also required between the two transactions. After a full data transaction, the slave can either continue to transmit without repeating the address bits, or the master can generate a NAK by pulling SDA HIGH when SCL is also HIGH, and end with a stop condition by pulling up SDA from LOW to HIGH when SCL is HIGH.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/i2c_protocol.png" width="400" height="70">
<figcaption align="center"> Figure 2: I2C protocol specified by MCP3221 </figcaption>
</center>
</div>


&nbsp;&nbsp;&nbsp;&nbsp;Following the I2C protocol explained above, we developed an 11-state FSM in SystemVerilog which can be seen in Figure 3. In STATE_DATA_RECEIVED_UPPER and STATE_DATA_RECEIVED_LOWER, the master receives data from the slave and puts it in a shift register of 12-bits. At each negative clock edge in these two states, the respective counter will increment by 1 and state transition happens when the counter reaches 7. Because the SDA line can be controlled by both the master and the slave, we wrote a tri-state buffer to decide which device gets control of the line. The tri-state buffer comes with an enable signal that is HIGH when the master device acts as an input in the transaction, and SDA should become high impedance on the master side when this enable signal is asserted. Otherwise, SDA gets the output from master. The SCL clock line follows the input 200K clock signal whenever we're not sending start/stop condition, not waiting, or not in idle. A data valid signal is pulled HIGH when the master acknowledges the lower bits of data or when a NAK is asserted by the master. This signal will later be used to control our integrator in calculating accumulative energy.

&nbsp;&nbsp;&nbsp;&nbsp;After figuring out the state machine, we tested our implementation by writing a testbench that feeds in control signal, clock, and data, and verified the output waveforms in ModelSim. As sow in Figure 4, the master is able to send out the correct address bits, generate ACKs when appropriate, read the input data and pull the data valid signal high when the 12-bit transaction completes.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/modelsim_waveform.jpg" width="400" height="70">
<figcaption align="center"> Figure 4: Modelsim waveform of testbench for I2C master implementation </figcaption>
</center>
</div>

