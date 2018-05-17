+++
title = "Visualizing Strange Attractors"
date = 2017-11-22T13:40:09+02:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["OpenGL"]
categories = []

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++

# What is a strange attractor?

The [wikipedia article](https://en.wikipedia.org/wiki/Attractor#Strange_attractor) on attractors gives the following definition/explanation:

> An attractor is called strange if it has a fractal structure. This is often the case when the dynamics on it are chaotic, but strange nonchaotic attractors also exist. If a strange attractor is chaotic, exhibiting sensitive dependence on initial conditions, then any two arbitrarily close alternative initial points on the attractor, after any of various numbers of iterations, will lead to points that are arbitrarily far apart (subject to the confines of the attractor), and after any of various other numbers of iterations will lead to points that are arbitrarily close together. Thus a dynamic system with a chaotic attractor is locally unstable yet globally stable: once some sequences have entered the attractor, nearby points diverge from one another but never depart from the attractor.

So for a dynamical system, which is usually a solution to a system of differential equations, we have some solutions which are called attractors. The attractors, attract a particle, or something moving in the system, meaning that when the particle falls into the attractor, it stays there. A relatable example is an asteroid that is moving in space and then starts orbiting a planet, and is then stuck in orbit, which is an attractor.

Some simple equations, like the [Lorenz system](https://en.wikipedia.org/wiki/Lorenz_system) are chaotic. This means that if you change the initial conditions slightly, what you get out after a certain number of steps, (you iterate the solution in steps usually), is going to be vastly different. When scientists model something, this is not a very attractive property. We would like our system to be robust to minor initial perturbations, but this is not always possible. Many physical systems that we model have this chaotic nature, most notably the weather, it is very hard to make weather forecasts far into the future with good accuracy. 

Although attractors are commonly associated with solutions of differential equations, they can also appear in general with an iterative system. An iterative system is a function, or a tuple of functions, where we iteratively feed the output in as input and *iterate* the solution in this way. This is also how you numerically find solutions to differential equations, where no analytical solution exists, or we can't find one.

In case I have piqued your interest in dynamical system, I took some [lecture notes](http://www.imm.dtu.dk/~guei/linfell.pdf) in a course I took some years ago. These notes are not quite finished, but should give you a rough idea about the basics and where to go from there.

# Why is this post tagged as Graphics?

Ok, so the point is that these attractors often look very nice. You can render them, but it can be a bit heavy computationally. I saw a very cool tweet by [Thomas Lin Pedersen](https://www.data-imaginist.com/) where he was rendering particles in a very nice way in R:

<center>
<blockquote class="twitter-tweet" data-lang="en"><p lang="und" dir="ltr"><a href="https://twitter.com/hashtag/generative?src=hash&amp;ref_src=twsrc%5Etfw">#generative</a> <a href="https://twitter.com/hashtag/rstats?src=hash&amp;ref_src=twsrc%5Etfw">#rstats</a> <a href="https://t.co/h4vDmbrOX8">pic.twitter.com/h4vDmbrOX8</a></p>&mdash; Thomas Lin Pedersen (@thomasp85) <a href="https://twitter.com/thomasp85/status/912754660867497984?ref_src=twsrc%5Etfw">September 26, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</center>

At that point I was inspired to do some cool visualizations myself, and learn how to make these aesthetically pleasing renderings. I also wanted to be able to make gifs or screen savers with these things.

I found [this](https://fronkonstin.com/2017/11/07/drawing-10-million-points-with-ggplot-clifford-attractors/) post about rendering strange attractors in R, using Rcpp. The Rcpp code for generating the points is actually very fast, what is the bottleneck here is the rendering with ggplot. So if you implement a rendering stack for a gif with this, the development time when you need to tweak many parameters is going to be rather slow.

# The Clifford attractor

So the attractor in the post I linked to above is called a Clifford attractor. What is interesting about the way it is generated is that the shape is controlled by four parameters $a,b,c$ and $d$. The shape is in some sense continuous in these parameters, meaning that if I make small adjustments in the parameters, the shape will not change too much. With this in mind I wanted to make a gif with it, I.e. we make a loop in the parameters space and render images for equally spaced parameters along this curve. For a small enough spacing, this will give me a continuously deforming shape and I will be able to loop it, because the curve in parameter space is a loop.

Not all parameters give us nice looking attractors, so I need a way to iterate this process fast, for different sets of parameters, to see what works. I talked to some of my colleagues that know about computer graphics, and they said that they way to go forward would be to use openGL. I actually found a tutorial of exactly what I needed [here](https://nathanselikoff.com/training/tutorial-strange-attractors-in-c-and-opengl). This is a tutorial on how to render multiple points, with point smoothing and good anti-aliasing, and this guy actually uses the Clifford attractor, **nice!**. What I was missing was ways to save intermediate plots, then I was just going to use ImageMagick to combine it into a gif. 

[The highest voted answer in this SO thread](https://stackoverflow.com/questions/3191978/how-to-use-glut-opengl-to-render-to-a-file) had everything I needed for saving the images. Now I just had to combine these things together!

After saving the images I used the bash command `mogrify` to change them into png images and finally ImageMagick to create the gif:
```bash
mogrify -format png *.*
convert -delay 120 -loop 0 *.png animated.gif
```

Here you can see the results on imgur:

<center>
<blockquote class="imgur-embed-pub" lang="en" data-id="a/ZFpHB"><a href="//imgur.com/ZFpHB">Strange attractors</a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>
</center>

[The code is also now on github](https://github.com/gumeo/movingAttractor) in case you want to play around with it!
