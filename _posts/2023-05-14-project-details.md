---
layout: post
title: FPGA based power estimator
featured-img: post_background
mathjax: true
---

## Introduction

Power consumption is a crtical consideration of a lot of the technologis today. Phone producers, for example, always talk about their improvement in power managemnet and how it makes their battery last longer. The balance between functionality and power is an intersting topic investigated by many. When we are doing labs, however, we rarely think about how much power we are consuming, because we are never limited by it. When we synthesize the giant drum in lab 3 for this class, we are exhasuting all DSP blocks on the FPGA, and we used a lot of M10K memory. This kind of extensive use of hardware is likely to result in high power consumption, and we are curious to find out what are some of the factors that could impact the power consumption of an FPGA. With this goal in mind, we built a circuit that measures the current flowing through the power cable, and implemented an integrator in the FPGA that continuously increments the enery consumed. Using these information, We drew three graphs displaying current, dynamic energy, and average energy on a VGA display. 

<div>
<center>
<img src="https://404codercn.github.io/ece5760_final_webpage//assets/img/posts/intro_pic.jpg" width="600" height="150">
<figcaption align="center"> Figure 1: Display of graphs on VGA </figcaption>
</center>
</div>

## Design & testing methods
In order to get current information about our FPGA, we need a current sensor that can read the current flowing through it and transmit this data to the FPGA. WIth some research online, we decided to go with the MIKROE-2987 current measurement board. This device relies on hall effect to measure the current passing through the internally fused input pins, and it has an extremely low series resistance of 1.2m &#937, which makes it an almost perfect ammeter. The output of this current sensor ges to a 12-bit ADC with an I2C interface. So in order to read the data stored in this device, we have to stablish I2C communication between it and the FPGA. 


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
