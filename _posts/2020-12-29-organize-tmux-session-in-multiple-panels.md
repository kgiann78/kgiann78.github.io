---
layout: post
title:  "Organizing a tmux session in multiple panels"
date:   2020-12-29 11:21:01 +0200
categories:
tags: [tmux, bash]
---

# Organizing a tmux session in multiple panels

When I work with **bash** in general, I like creating scripts even for simple things. This is mostly because I tend to forget things but also in case I need something, either use the old script or look it up and create a new one.

In that way, when I started using [tmux](https://github.com/tmux/tmux/wiki) I had to write down some basic commands otherwise I would definitely forget them. But in the process something very useful came out.

So, I had this thing where I should fire up a number of services in order to have the main application up and running. I could do that with plain **bash** of course but I wanted to use tmux, 
in order to have the whole process under one session. 
Moreover, I wanted to utilize its panels so that I could have everything in one terminal window. Btw, I am using an Ubuntu terminal app (not Terminator or anything else that could create multiple windows/panels).


### First part, create the session

Long story short, creating a tmux session is as simple as that:

    tmux new-session -s ses-0 -n my-screen-name -d

where `-s` defines the name of the session (this is also needed when we want to kill the session). and with `-n` is the session's screen name.

### Part 1b, kill the session

It would be much easier to say at this point that, whenever we want to kill a tmux session we can do this with the following command:

    tmux kill-session -t ses-0

where `-t` is targeting the session we want to kill. With this command we kill all opened programs that are currently executing within our tmux session.


### Second part, split panels

From here on we can split the screen vertically, horizontally and whatever combinations we can do. 
The first time we split the screen (either way) the first half will be `0` and the second `1`. 

    tmux split-window -v
    tmux attach-session -t ses-0

The screen will be divided in half unless we say otherwise. 

🏆 Note that:
Despite it is a bit early to explain, use line

    tmux attach-session -t ses-0

After the `split-window` command in order to see the result in the screen. Just make sure that the target argument `-t` points to your tmux session.


![tmux_vertica_split](/assets/img/tmux_vertical_split.png){: .mx-auto.d-block :}


From here on, if we add another split, without defining which panel to split, it will automatically split the second one (panel `1`):

    tmux split-window -v
    tmux split-window -h
    tmux attach-session -t ses-0

![tmux_horizontal_split_1](/assets/img/tmux_split_horiz_panel_1a.png){: .mx-auto.d-block :}


We can point out though which panel we want to split with the `target` argument:

    tmux split-window -v
    tmux split-window -h -t 0     # here we are targeting panel 0
    tmux attach-session -t ses-0

![tmux_horizontal_split_0](/assets/img/tmux_split_horiz_panel_0a.png){: .mx-auto.d-block :}


### Executing commands in our panels

After we finalize our window separation, we can now tell tmux to execute something in each panel. Just remember the above and note what is each panel's id to target.

As an example we will create 3 panels, one in the top half of the window and two at the lower half, splitting it in two horizontally. At thouse 3 panels we will run the following:

* In panel `0` (top half) we will execute htop  
* In panel `1` (lower left panel) we will run a simple bash script that will print a timestamp every 5 seconds
* In panel `2` (lower right panel) we will run a plain `ls -la` of the `/tmp` directory

And do as follows:

    # split window vertically to panels 0 and 1
    tmux split-window -v 

    # split panel 1 horizontally to panels 1 and 2 (left to right)
    tmux split-window -h 

    # execute htop
    tmux send-keys -t 0 "htop" C-m

    # run datetime script
    tmux send-keys -t 1 "while true; do echo $(date --iso-8601="seconds"); sleep 5; done" C-m

    # run ls -la /tmp
    tmux send-keys -t 2 "ls -la /tmp" C-m

    tmux attach-session -t ses-0


And here is the result:

![tmux_executing_commands](/assets/img/tmux_executing_commands.png){: .mx-auto.d-block :}


### Closing

After creating all those panels we can navigate with tmux's `ctrl-b` `+` `arrow-keys` or what is your navigation configuration.

As next steps we could fine-tune the size of each panel. Right now every time a split occurrs it splits the target panel in half. 
It would be better though to be able to set the desired width or height at that time.

I hope you will find this useful, future me. If you think this approach is old or redundant please let me know.

Cheers!
