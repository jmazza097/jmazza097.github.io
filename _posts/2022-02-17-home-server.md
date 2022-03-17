---
title: Home Server with Raspberry Pi
aside: true
categories:
- Raspberry Pi
- Home Server
feature_image: "https://picsum.photos/2560/600?image=872"
---

Home Server Project using Docker and a Raspberry Pi Part 1:
<!-- this part ^^ is how much shows in the description of the post by using a parachgraph format it autmoatically pics how much to show -->
<!-- more -->

This is my first blog post on my new site, and I am excited to see where the blog goes. Also, to preface, I am not the greatest writer, so I am sorry for any egregious grammar mistakes. Now to the content! I was recently gifted a Raspberry Pi 3 from my good friend Michael Lamb. Right away, I knew I wanted to get it set up running Pi Hole. In this article, I will discuss what I installed on the Pi to get a solid base setup. This will make it easier to run and maintain the Pi Hole once I get to that step. I used the Pi imager to install [RASPBERRY PI OS LITE](https://www.raspberrypi.com/software/) a port of Debian Bullseye with no desktop environment onto the SD card. Once that was installed I got the SD card into the Pi. I connected it to the Internet via an ethernet cable from my unmanaged switch in my office.

{% include figure.html image="/assets/pictures/raspi.jpg" width="700" height="900" %} Once the Pi was online, I grabbed its IP off my router by going to 192.168.0.1. With my Pi’s IP, I used Putty to SSH into the Pi. The default login from these images is Username: pi and Password: raspberry. Once in, Docker was the first thing to install. Docker is a lightweight tool for running applications in containers that are similar to virtual machines. This is very important when using a Pi because it is resource-limited. I followed [this guide to install Docker](https://dev.to/elalemanyo/how-to-install-docker-and-docker-compose-on-raspberry-pi-1mo) through step 6. With Docker installed, Portainer was next. Portainer provides a friendly GUI to view all of the containers and use Docker without using the command line. Installing Portainer on the Pi is very straightforward. Here is another [guide](https://pimylifeup.com/raspberry-pi-portainer/) with the commands to input into the command line. With Portainer installed, I set up a username and password as it does not come preconfigured. Then select Docker as the program you want to pair Portainer with, and once that is taken care of, the home screen should give a view of your local docker instance. Here is mine with a few containers on it.
{% include figure.html image="/assets/pictures/portainer.png" width="900" height="1100" %} Now that I have a solid base for the Pi, I can start adding containers with various programs that I will discuss in later posts.