---
layout: post
title:  "Installing glassfish in ubuntu VM at ~okeanos or Digital Ocean"
date:   2017-01-29 22:30:01 +0200
categories: docker glassfish
---
This is the first part of a series of steps I took to build an Enterprise Application in Java!

So, to begin with, the most basic step is to have an application server. A server that can handle enterprise applications (ear) quite well is Glassfish 4. This server needed to be installed in a Ubuntu VM at ~okeanos. After a quick search I found this tutorial <a href="https://www.digitalocean.com/community/tutorials/how-to-install-glassfish-4-0-on-ubuntu-12-04-3" target="_blank">How To Install Glassfish 4.0 on Ubuntu 12.04.3</a>.

Prior to this I had already installed a glassfish application server in my machine (Mac OS X) really easy, just by downloading <a href="http://download.java.net/glassfish/4.0/release/glassfish-4.0.zip">glassfish4</a> and unziping the zip file where I wanted to. Don't forget to update the PATH in .bash_profile.

Starting a domain or deploying war files is already mentioned in the linked tutorial above. In future posts I will talk about creating enterprise applications with <a href="https://maven.apache.org">Maven</a> and adding Maven glassfish plugin to interact with the currently installed glassfish server or manually deploy ear files in glassfish.
