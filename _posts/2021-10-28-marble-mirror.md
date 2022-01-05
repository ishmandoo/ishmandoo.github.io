---
layout: post
title: Marble Mirror (work in progress)
categories:
- Electronics
- Projects
- Programming
tags: [WIP]
comments: []
---


![Photo front](/assets/img/2021/10/marble_mirror_front.png){: .center width="95%"}


I've always sort of wanted to make a marble-based machine. I've been following the progress of Wintergatan/Martin Molin's [Marble Machine X](https://www.youtube.com/watch?v=C8qyVURtSZc) for some time. It's a follow-up to the original and more well-known [Marble Machine](https://www.youtube.com/watch?v=IvUU8joBb1Q) of YouTube. 

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/IvUU8joBb1Q" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

You should probably just leave that video running and let it be the background music while you read this post.

This time around, Martin's documented his progress with a compelling series of videos showing the ups and downs of his adventure to build a giant music playing machine based around marbles. 
A marble machine almost seems simple. It's just tracks and belts and xylophones and stuff. 
His videos put the lie to this idea.
The Marble Machine X project has stalled and as of late 2021, he's pursuing a ground-up redesign of his incredible machine after [coming to terms with some fundamental design flaws](https://www.youtube.com/watch?v=WN90HYiFpAw).

Fortunately, we picked a much easier project.
The name is ridiculous at this point, but it's stuck.
The original idea was to produce some kind display where the "pixels" are made of a grid of macro-scale physical objects and connect this screen to a webcam to create a "mirror".
The direct inspiration was [Daniel Rozin’s mechanical mirrors](http://www.smoothware.com/danny/).
<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/1ZPJ0U_kpNg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

We considered a few designs but settled on one that's nowhere near real time. We finally have it working:
<center>
<blockquote class="twitter-tweet"><p lang="fr" dir="ltr">A momentous occasion! <a href="https://t.co/fWjtwTHqRN">pic.twitter.com/fWjtwTHqRN</a></p>&mdash; Philip Zucker (@SandMouth) <a href="https://twitter.com/SandMouth/status/1453563957696999430?ref_src=twsrc%5Etfw">October 28, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</center>

It forms an image in the style of a Connect 4 board. A stepper motor lifts a column of marbles to the top of the machine by rotating a notched disk (blue) and drops one into the carriage (green). A RGB sensor in the carriage (red) measures the color of the cargo area and decides if it contains a reflective steel marble, a black marble, or no marble. If the marble can be used in one of the columns of the image, it is delivered to that column and dropped in. If the marble can't be used because it doesn't match the next pixel in any column, it is rejected and dropped back into the reservoir at the bottom.

![Model front](/assets/img/2021/10/model_front.png){: width="48%"}
![Model front](/assets/img/2021/10/model_rear.png){: width="48%"}

The carriage rides on linear bearings on an 8mm steel rod. It's driven by a stepper motor and lead screw. A servo with a plastic horn acts as a gate.


![Model front](/assets/img/2021/10/carriage.png){: width="38%"}
![Model front](/assets/img/2021/10/carriage_2.png){: width="58%"}