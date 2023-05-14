---
layout: page
title: About
permalink: /about/
---

This is the webpage of our ECE5760 final project. We built a power estimator that calculates the power consumption of an FPGA. We established I2C connection between the FPGA and a current sensor by following the protocol specified by the [ACS711 sensor](https://download.mikroe.com/documents/datasheets/ACS711-Datasheet.pdf). Since the voltage across the power adapter is always at 12V, we were able to calculate the unit energy using V x I x dt. Both dynamic and average power were provided in our design. Through testing, we were able to observe significant difference in power consumption between using ethernet cable and using serial to communicate with the FPGA. Read about our project to find out more details! 

You can view our code on [GitHub:](https://github.com/404coderCN/ECE5760_power_estimator)

<video width="600" height="400" controls>
  <source src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/vid_demo.mp4" type="video/mp4">
  <source src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/vid_demo.ogg" type="video/ogg">
Your browser does not support the video tag.
</video>




