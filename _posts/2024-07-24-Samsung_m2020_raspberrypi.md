---
layout: post
title: "Samsung M2020 on Raspberypi"
date: 2024-07-24 21:00:00 -0000
author: "TimDude"
categories: raspberrypi cups
tags: raspberrypi cups driver arm rastertospl
---


Well well well, Samsung / HPs uld driver does not play well with arm architecture.
The Solution we compile oour own drivers
## Install building software
~~~
apt install git g++ make libcups2-dev
~~~

## Download source
~~~
git clone https://gitlab.com/ScumCoder/splix.git
~~~

## Build the drivers
~~~
cd splix/splix
make DISABLE_JBIG=1
sudo  make install
~~~
