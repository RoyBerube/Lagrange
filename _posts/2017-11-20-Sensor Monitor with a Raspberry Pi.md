---
layout: post
title: "Sensor Monitor with a Raspberry Pi"
author: "Roy Berube"
categories: journal
tags: [raspberry pi, python ]
image:
  feature: nails2.jpg
  teaser: nails2-teaser.jpg
  credit: 
  creditlink:
---

So I had a Raspberry Pi not getting much use. I had been using it with OSMC as a video streaming device to watch on my TV, but the software was just too annoying to use for a number of reasons. So I turfed that software and installed Raspbian on the little device. The little computer needed a new purpose.

I have a sump in the basement that gets a lot of use when the ground is saturated, especially in the spring as snow is melting. A sump failure would cause significant water damage. Checking the sump manually on a regular basis was getting tiresome. There had to be a better way. I know there are sump alarms available commercially, but what is the fun in that? Raspberry Pi to the rescue. My goal was to use it as a monitor that would email me a message if the sump sensor ever activated. That way I could have peace of mind wherever I went with my cel phone.

Raspbian is on the Debian branch of Linux. Most importantly for my purposes, it has drivers to access the GPIO pins on the device. GPIO stands for *general purpose input-output*. The GPIO interface is perfect for measuring a switch with just two states. My sump sensor is a float switch with two states. A plan is taking shape.

Raspbian also comes with some useful software that is excellent for my purposes. *Node Red* provides a drag and drop interface for logic structure of your program. Reading a switch and sending an email with it is very easy. Another great feature is its web interface - useful to check that the device is running and to see the sensor reading. 

After some testing of the *Node Red* switch sensing input I was less enthusiastic about it. It uses edge detection and about 1 or 2 percent of the time would miss detecting a signal transition. With edge detection it only sends a signal again when the switch changes state again. Not good. There had to be a better way.

Luckily, Node Red also gives users an easy way to run an executable script. Python is perfect for this. I decided to use busy polling with the sensor being read at regular intervals - it is currently running every 30 seconds. Here is the script: 
```python
#!/usr/bin/python3
import sys 
import RPi.GPIO as GPIO
import time
# Intended to be run as a node-red script.
# This works as a busy poll method. No edge detection.
# More reliable than node-red GPIO input node.
GPIO.setmode(GPIO.BCM)
sensor_in = 21 
GPIO.setup(sensor_in, GPIO.IN, pull_up_down = GPIO.PUD_UP)
# Allow time for pin to stabilize. Only matters first pass after pi power on.
time.sleep(.05)
# Invert the signal. Only possible inputs are 0 or 1
ss = GPIO.input(sensor_in)
ss = (ss *-1) +1
# Any printed output is seen as a message payload. Simple.
print(ss)
```
There was only one issue. After a reboot it would send a false positive. That is the reason for the 50 millisecond delay *time.sleep(.05)* between setting up the GPIO pins and reading them. The pin needs time to achieve its new state. It only matters the first time the script is run after a reboot, but since the delay is insignificant it can be left in without detriment.

I am currently running the Pi headless (no monitor, keyboard or mouse) over my network. *Node Red* can be programmed with a browser over the network, so the graphical interface is still usable. Here is what it looks like:

![Node Red](/assets/img/NodeRedSensor.PNG)

The rate limiter node is there to limit the amount of emails sent. Without it, an email would be sent every time the Python script is run. I have it set to limit an email message once every 6 hours.

Here is what the web user interface looks like when deployed:

![Node Red User Interface](/assets/img/NodeRedOutput.PNG)

 This also displays the current temperature obtained with a web scraping Python script. When I check that the device is running it is handy to get the temperature at the same time. This pulls the temperature from Environment Canada's reading at Blatchford Field in Edmonton, Alberta. The **Refresh** button forces a web scrape to get the most current reading; the script is set to run automatically every 20 minutes so it might be a little out of date. The *Node Red* layout is quite simple:  
 
 ![Node Red Scraper](/assets/img/NodeRedScraper.PNG)
 
 Here is the temperature web scraper Python script:
```python
#!/usr/bin/python3
"""Scraping gc.ca weather xml files
for current temperature.
"""
from urllib.request import urlopen
from bs4 import BeautifulSoup as soup

scrape = urlopen(
    'http://dd.weatheroffice.ec.gc.ca/citypage_weather/xml/AB/s0000045_e.xml')
xml = scrape.read()
scrape.close()
parsed = soup(xml, "html5lib")
current = parsed.find("currentconditions")
temp = current.find("temperature").text
print(temp)
```

That is about it for this setup. To make it more reliable I run a cron job to reboot the device every night - I was having problems with the Pi locking up after a few weeks of running continuously.