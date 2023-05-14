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

&nbsp;&nbsp;&nbsp;&nbsp;We all know the equation $P = I * V$ from physics class, since we already know the power adapter of the FPGA delivers a constant 12V voltage, we need a way to get the current reading. With some research online, we decided to go with the MIKROE-2987 current measurement board. This device relies on hall effect to measure the current passing through the internally fused input pins, and it has an extremely low series resistance of 1.2m $\Omega$, which makes it an almost perfect ammeter. The output of this current sensor goes to a 12-bit ADC with an I2C interface. So, to read the data stored in this device, we have to establish I2C communication between it and the FPGA. We achieved this by developing a finite state machine that adheres to the I2C protocol specified by the [MCP3221 ADC](https://www.digikey.com/htmldatasheets/production/48559/0/0/1/mcp3221.html?utm_adgroup=Integrated%20Circuits&utm_source=google&utm_medium=cpc&utm_campaign=Dynamic%20Search_EN_Product&utm_term=&utm_content=Integrated%20Circuits&gclid=CjwKCAjwjYKjBhB5EiwAiFdSfp9ML477NVziVD5azNRU4RDcGAhePigK16BnE5FcO95hGj4eKhzXJRoCMEwQAvD_BwE). The I2C communication only involves 2 wires-- SDA for data, and SCL for clock. In our case, the SCL line is controlled by the master and the SDA line can be controlled by either the master or the slave depending on which state the system is in. According to the datasheet, each transaction between the master and the slave should be 8 bits long and is always followed by an ACK/NAK. To initiate a transaction, the master device has to signal the start condition by pulling down the SDA line from HIGH to LOW while SCL remains HIGH. Then, the master should send out 4-bit device bits followed by 3-bit address bits, default to 1001101, and a read/write bit where 1 represents READ and 0 is WRITE. It's worth noting that changes in the SDA line should always happen when SCL is LOW, remain stable at the HIGH portion of the clock (except start and stop condition), and there should be one clock pulse per bit of data. After the initial transaction of address, the slave device will respond with an ACK that signals it has received the data. To send an ACK, a device must pull the SDA line LOW at the HIGH portion of the 9th pulse of SCL. The slave device, in this case the MCP3221 ADC, can then start sending the data stored in its internal 12-bit register. Since each transaction is limited to 8-bit, each valid current reading takes two transactions with the first four bits always equal to 0000. An ACK from the master is also required between the two transactions. After a full data transaction, the slave can either continue to transmit without repeating the address bits, or the master can generate a NAK by pulling SDA HIGH when SCL is also HIGH, and end with a stop condition by pulling up SDA from LOW to HIGH when SCL is HIGH.

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/i2c_protocol.png" width="400" height="70">
<figcaption align="center"> Figure 2: I2C protocol specified by MCP3221 </figcaption>
</center>
</div>

1. Just download or fork and clone the source from [github.com/janczizikow/sleek](https://github.com/janczizikow/sleek/).
2. Make sure your local machine has ruby and node
3. Edit site settings in  `_config.yml` file according to your project.
4. Replace `favicons` and `_includes/logo.svg` with your own logo.

**Note** that you might have to adjust some CSS depending on the width and height of your logo. You can find Header / Navigation related SCSS in `_sass/layout/nav.scss`.

## Writing content

### Posts

Create a new Markdown file such as `2017-01-13-my-post.md` in `_post` folder. Configure YAML Front Matter (stuff between `---`):

```yaml

---
layout: post # needs to be post
title: Getting Started with Sleek # title of your post
featured-img: sleek #optional - if you want you can include hero image
---

```

#### Images

In case you want to add a hero image to the post, apart from changing `featured-img` in YAML, you also need to add the image file to the project. To do so, just upload an image in `.jpg` format to `_img` folder. The name must before the `.jpg` file extension has to match with `featured-img` in YAML. Next, run `gulp img` from command line to generate optimized version of the image and all the thumbnails. You have to restart  the jekyll server to see the changes. Sleek uses [Lazy Sizes](https://github.com/aFarkas/lazysizes) Lazy Loader for loading images. Check the link for more info. Lazy Sizes doesnt't require any configuration and it's going to be included in your bundled js file.

### Pages

The home page is located under `index.md` file. To change the content or design you have to edit the `default.html` file in `_layouts` folder.

In order to add a new page, create a new html or markdown file under root directory or inside `_pages` folder. To add a link to the page, edit `navigation` setting in `_config.yml`.

### Images TODO

Introduce gulp optimization

Breakpoint | Image Type | Width | Retina
------------ | ------------ | ------------- | -------------
xs |Post Thumb | 535px | 1070px
sm |Post Thumb | 500px| 1000px
md |Post Thumb | 329.375px | 658.75px
lg |Post Thumb | 445.625px | 891.25px
xl |Post Thumb | 353.125px | 706.25px

Breakpoint | Image Type | Width | Retina
------------ | ------------ | ------------- | -------------
xs |Post Hero | 535px | 1070px
sm |Post Hero | 500px| 1000px
md |Post Hero | 329.375px | 658.75px
lg |Post Hero | 445.625px | 891.25px
xl |Post Hero | 353.125px | 706.25px

### MathJax

If you want to use [MathJax](https://www.mathjax.org/) in your posts, add `mathjax: true` in [YAML front matter](https://jekyllrb.com/docs/frontmatter/) of your post:

```yaml
---
layout: post
title: Blog Post with MathJax
featured-img: sleek # optional - if you want you can include name of hero image
mathjax: true # add this line in order to enable MathJax in the post
---
```

#### Example

In N-dimensional simplex noise, the squared kernel summation radius $r^2$ is $\frac 1 2$
for all values of N. This is because the edge length of the N-simplex $s = \sqrt {\frac {N} {N + 1}}$
divides out of the N-simplex height $h = s \sqrt {\frac {N + 1} {2N}}$.
The kerel summation radius $r$ is equal to the N-simplex height $h$.

$$ r = h = \sqrt{\frac {1} {2}} = \sqrt{\frac {N} {N+1}} \sqrt{\frac {N+1} {2N}} $$
Happy hacking!
