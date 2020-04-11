# Smart Home Overview

The below is a big-picture overview of the details of my smart home setup. The code in this repo makes up the majority of the config for Home Assistant, but this Readme aims to provide a general overview of the system as a whole, so that one can better understand the context of this config.

### Hardware

Smart home setup is run from an Intel Nuc ([Wilson Canyon](https://ark.intel.com/content/www/us/en/ark/products/76977/intel-nuc-kit-d54250wyk.html)) in my “media closet.” Nuc is running Ubuntu 18.04 plugged into battery backup with a [Nortek HUSBZB-1](https://www.nortekcontrol.com/products/2gig/husbzb-1-gocontrol-quickstick-combo/) Z Wave / Zigbee dongle. I don’t use a monitor on this machine, so will usually access it from my laptop using OpenSSH. I’ve found VNC and Chrome Remote desktop to work too, but working through the UI is usually more trouble than it’s worth. However, more than one access option is a nice safeguard for a moderately experienced person like myself, who doesn’t like setting up a keyboard and monitor in the closet when I need to fix stuff (has definitely never happened…).

I have a variety of sensors, cameras, and IOT integrations that connect through Wifi, Z Wave, and/or Zigbee. Where possible, I try to steer away from Wifi, as network interference in 2.4 ghz is already a problem in our dense neighborhood.  Since Z Wave and Zigbee operate mesh networks on different frequencies, they may be more reliable over the long term.  Zigbee hardware is cheaper generally (check out Ziaomi Aqara for example), but for some items, such as light switches, there are just more good options with Z wave. Hence the desire to use both protocols. 

### Home Assistant General Info

[Home Assistant](https://www.home-assistant.io/hassio/) is an open source home automation platform that runs as a disk image on Linux-based platforms (common is Raspberry Pi), or can be run using Docker on more powerful machines. I chose this platform due to its unmatched flexibility compared to the competition (such as Smarthings, Wink, etc.) and its negligible cost.  It is also a certifiably nerdy hobby, which has kept me entertained for months on end.

I am technically running Home Assistant “Supervised” (previously called Hassio) in a series of Docker containers (see here: once docker is installed, run the instructions for “[Generic Linux Host](https://www.home-assistant.io/hassio/installation/#alternative-install-home-assistant-supervised-on-a-generic-linux-host/”).  This methodology allows you to add/remove addons from within the UI; each creates its own Docker container, and using Docker Compose is not required, even for providing HA access to my USB dongle. If you use the install “[Home Assistant Core in Docker Container](https://www.home-assistant.io/docs/installation/docker/)” methodology, then you’ll be installing addons outside of the UI, and Docker Compose is recommended. The latter path seems to be more of a best-practice, but I have not yet felt compelled to try it.

My objectives with this system are two-fold: security and convenience.  The security part was really the impetus to get started here. My wife and I moved into a new house, and wanted to get some sort of security system in place, but looking at the options frustrated me because of the cost and/or somewhat questionable reliability.  So rather than pay someone way too much money to install a mediocre system, I figured why not install my own slightly less mediocre system for a bunch less money?  The convenience part has just been cool, an example being that I have an inordinate amount of time perfectly orchestrating the logic of our lamp in the living room, so that it is always on when we need it, and always off when we don’t.  

I won’t go into detail with all of the specific automations here, but there are some worth checking out in this repo.  Can you figure out what happens when the alarm is “armed” and a door is opened?

### Network Video Recorder (NVR): MotionEye

I tested out multiple options for a Linux-based NVR: Zoneminder, Shinobi, and [MotionEye](https://github.com/ccrisan/motioneye/wiki).  I spent a month or so running Zoneminder, and was initially very happy with it, mostly due to how advanced the motion detection configuration options are, as well as ZM Ninja being a super cool mobile app, but over time, I began having struggles with cameras going down. After trying many things to get the reliability up, I ended up “shopping around,” and in the end, I landed on MotionEye. It’s much simpler than Zoneminder, and isn’t 100% there with the features, but the interface is great and it’s been super reliable for me running four cameras.

MotionEye installs directly into Ubuntu, it can be installed into Docker as well, but for a stable application like this, I so far have not found a compelling reason to add the additional complexity of using Docker here (if you have a good reason, please let me know!).  

My four cameras each send an RTSP stream, so I enter each of their addresses into MotionEye, which include the login creds for those cameras, and I’m off and running.  I set up recording on my two outdoor cameras, motion-triggered only.  So I’ve spent some time dialing in the triggers, such that I have a manageable amount of recordings to sift through at any given point.

Motioneye has no mobile app, but it looks great in a mobile browser, so I just saved the page to my phone home screen and it works great. It also auto-generates local video stream url’s for each camera, so I’ve added those in as iframe elements in Home Assistant.  Now if the alarm were to be tripped, I could click my Home Assistant push notification that came through (iphone Critical Alert specifically), and launch straight into HA to see the live camera feeds.


### Mobile Access/VPN

When on my phone, at home on the local network, the Home Assistant app and Motioneye in the browser work great, but OOTB, neither of these are going to work when off of our home wifi.  There are multiple options for enabling this, such as HA has a subscription cloud service that allows this seamlessly, but of course this wouldn’t give me access to MotionEye recordings.  Port forwarding through the router is always an option, but this is my security system for Pete’s sake!  Anyway, I landed on the fact that the only way to do this right, is to use my own VPN server.

In order to achieve that, I installed OpenVPN on my Nuc (my own slight variation of [these instructions](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-18-04) at Digital Ocean), set another computer as the certificate authority (CA), and in no time at all, I had an active server with a signed certificate and key. I created two additional certificates, and signed them with my CA, one for each phone. After installing the OpenVPN app in each phone, I generated .ovpn files using the certs and keys on the server, sent them to the phones, and I was then able to activate them as clients in the OpenVPN app.  

So now my VPN server has two clients. It was a bit tricky setting up the configuration, and there are still some issues that could pop up. Such as, if I knew better early on, I would have set up my home router IP range to be something more strange. One of these days, I’ll be on wifi somewhere and the network will contain a device with the same IP as my Nuc at home (oops). I hope I don’t have a “need to see what trouble the cats are getting into” emergency at that exact moment!

### Version Control, CI, and almost/sort of CD

Once I got my Home Assistant config generally setup, I wanted a better way to edit/update my code changes without risking blowing up my entire system (happened to me plenty of times before I learned). So I landed on using Github to maintain a Master repository remotely and work with Git to make any changes to local branches first. There are still plenty of aspects of HA that are configured within the “production” UI, such as the Dashboard layout (e.g. Lovelace) and sensor setup using ZHA (Zigbee) and Z Wave, but most of the aspects that could really make or break server up-time are controlled by .yaml files that I manage with Git.

First step was to push my config up to Github using [these](https://www.home-assistant.io/docs/ecosystem/backup/backup_github/) instructions.  Once my remote repo was set up, I created a Staging branch and cloned that locally on my laptop, which I branch off of locally to work before pushing changes back up to the remote. Once I had that up and running, I got the idea to use Continuous Integration testing, so that I can feel confident in my config before it commits to the Master, and is subsequently pulled down to the HA server.  

I used Travis CI to spin up two test builds for Home Assistant (“latest” and “beta”) in a remote testing environment.  The integration between Travis and Github works great.  I also created an auto-merge script that runs remotely after CI test pass, notification of build results email, and a button + bash script in HA to do a Git pull from origin Master whenever I hit the button.

Current workflow looks like this:

1) Edit and make commits to local Development branch
2) Once ready to deploy feature, merge with local version of Staging branch, which is kept up to date with remote Staging branch.
3) Push local changes from local Staging to remote Staging
4) Once new commits are received in Staging branch, Travis CI builds begin
5) If both builds pass (still working on using staged builds correctly with Travis), merge changes with remote Master repo. Current state: merge occurs if any builds pass.
6) Email fires off to report build results, if they passed, I know the merge is complete. If not, I don’t need to worry about Master being affected by my recent push.
7) At any point I’m ready to update HA server config, I open the App on my phone or laptop and click my ‘Get Latest Config’ button.
    a) I thought about using pull requests, and various other git permutations, like rebasing, but overall have discovered with one contributor from one machine, it just kinda works.  Please excuse my horrendously messy commit history though, I finally figured out that I should test my CI build stuff in temporary branches only….
