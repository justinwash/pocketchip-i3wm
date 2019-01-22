# i3wm-chip
i3wm configuration for the PocketCHIP

An updated guide for installing i3 window manager on pocket C.H.I.P
Though I love my Pocket C.H.I.P. and realize that it's more or less the only product of its kind (handheld full Linux device with a physical keyboard and decent battery life) I've never found myself to be happy with PocketHome. I frequently work with CLI-based tools, and having to navigate around the rather elementary and clearly PICO-8 oriented interface wasn't really doing it for me. So, I decided to set out to configure my Pocket C.H.I.P. in a more professional, traditional Linux fashion. A guide exists for this already, but not only does it have to be viewed from an archive, but also, I found it to be outdated, as well as non-functional in some places. So here's my shot at making a better tutorial.

One thing to remember is the concept of the "mod" key, which is central to navigating your desktop in i3wm. In this guide, the mod key will always be referring to ALT.

Getting Ready

To start, we'll be installing i3 itself, as well as getting things ready to install a tool for running full desktops on the Pocket C.H.I.P., PocketDesk, afterwards.

sudo apt get-update

sudo apt-get upgrade

sudo apt-get install i3 i3blocks git

Installing an SSH server on the Pocket C.H.I.P. (Optional)

If you'd like to make this process ten times easier for you, I strongly suggest setting up your Pocket C.H.I.P. for SSH. To do this:

sudo apt-get install openssh-server

Once this is done, you're free to use an SSH client on a laptop or desktop to connect to your Pocket C.H.I.P. to enter commands remotely. To connect from another Linux device, make sure you have OpenSSH client installed, then enter the following command into the terminal of the client device:

ssh chip@192.168.1.X

with chip being your current username, and X being the last digit of your Pocket C.H.I.P.'s current IP address locally, for instance, "192.168.1.5". If your network is configured differently than the norm, then it's possible you'll have to adjust other values in the IP, but for most people, just changing the last number should work just fine.

If you're on a Windows machine, you can either install putty or use the native OpenSSH client for Windows 10, then follow the above steps.

Back to business: Installing PocketDesk and getting started with i3wm

Once that's finished, we'll be installing PocketDesk. This is helpful because it gives the user the option to run the full CHIP desktop image, as well as making sure the keyboard, touchscreen, and such function properly in environments other than PocketHome. Installing this is pretty easy, just run the following commands:

git clone https://github.com/AllGray/PocketDesk.git

sudo ./PocketDesk/PocketDESK.sh

After this, go ahead and reboot your chip, and you'll be presented with a login screen. By default, this screen will log out into PocketHome. However, you can change which environment you would like to log into by hitting ESC, then left on the "dpad" twice to get to the window manager/desktop environment menu. Use the dpad to scroll to whichever you would like to boot into, in our case "i3", hit enter. Enter your credentials (username "chip" and password "chip" by default, however, you really should change this as soon as you get the chance) then hit enter again. This should successfully drop you into a session of i3. Feel free to follow the setup prompt to configure your i3 so you can get a feel for it, however, we'll just be replacing it with a new config, so don't worry about it too much.

Configuration of i3wm

First, we'll need to make a script for changing brightness, because this option isn't easily available in i3 with our current config. Go ahead and

cd /bin

So we're in the folder where we'll be storing the script. Now open a new text file using your favorite text editor, although nano is generally a good go-to for beginners:

sudo nano brightness

In this file, enter the following: (I strongly recommend using SSH to do this, it's not technically necessary, but being able to paste these things in is going to make your life so much easier)

 #!/bin/bash

sysfs="/sys/class/backlight/backlight"
max=`cat ${sysfs}/max_brightness`
level=`cat ${sysfs}/brightness`

usage()
{
script=${0##*/}
echo
echo "Invalid usage of ${script}!"
echo "  $1"
echo "$script up     : increases brightness"
echo "$script down   : decreases brightness"
echo "$script set #  : sets brightness to # (integer)"
echo

exit 1
}

set_brightness()
{

level=$1

if [ $level -lt 1 ] ; then
 level=1
elif [ $level -gt $max ] ; then
 level=$max
fi
 
echo $level > $sysfs/brightness 
}

case "$1" in
  up)
    let "level+=1"
    set_brightness $level 
    ;;
  down)
    let "level-=1"
    set_brightness $level 
    ;;
  set)
    if [[ ! $2 =~ ^[[:digit:]]+$ ]]; then
     usage "second argument must be an integer"
    fi

    set_brightness $2
    ;;
  *)
    usage "invalid argument"
   esac
Next, we'll make another script in /bin since we're already here. This one will be called battery, and will deal with getting an (admittedly somewhat rough) battery charge percentage from the Pocket C.H.I.P.'s hardware.

sudo nano battery

Input the following into the file, and save:

VOLTAGE_DROP=69
MAX_VOLTAGE=4214
MIN_VOLTAGE=3600
is_charging=$(cat /usr/lib/pocketchip-batt/charging)
voltage=$(cat /usr/lib/pocketchip-batt/voltage)
voltage_offset=$(bc <<< "$is_charging*$VOLTAGE_DROP")
excess_voltage=$(bc <<< "$voltage-$voltage_offset-$MIN_VOLTAGE")
max_excess_voltage=$(bc <<< "$MAX_VOLTAGE-$VOLTAGE_DROP-$MIN_VOLTAGE")
percentage=$(bc <<< "scale=2; $excess_voltage/($max_excess_voltage/100)")
status="-" && [[ "$is_charging" == 1 ]] && status=" ^ ^ "
echo $percentage$status
For more info on how to edit this script to be more accurate to your battery's condition, see this post.

Now we'll make sure these scripts have the proper permissions with:

sudo chmod 775 brightness && sudo chmod 775 battery

And we're done working with those scripts.

By default you'll notice that the status bar on the bottom of the screen (known as the i3bar) doesn't fit well on the screen, and some of the information it's trying to display isn't really functioning properly. So let's replace this status bar with one tailored to the Pocket C.H.I.P.'s hardware. To get started, change to the /etc directory:

cd /etc

and open a new text file called "i3blocks.conf"

sudo nano i3blocks.conf

This is a pretty simple file to customize if you so choose, but for right now, let's start with a simple baseline that displays internet connection status, battery charge level, and the date and time. Paste the following content into i3blocks.conf:

# i3blocks config file
#
# Please see man i3blocks for a complete reference!
# The man page is also hosted at http://vivien.github.io/i3blocks
#
# List of valid properties:
#
# align
# color
# command
# full_text
# instance
# interval
# label
# min_width
# name
# separator
# separator_block_width
# short_text
# signal
# urgent

# Global properties
separator_block_width=10

[wireless]
label=W
instance=wlan0
#instance=wlp3s0
command=/usr/share/i3blocks/network
color=#00FF00
interval=10

[battery]
#label=BAT
#label= ^
command=sh /bin/battery
color=#F2FF00
interval=15

[time]
command=date '+%Y-%m-%d %H:%M:%S'
interval=5
Now that both necessary scripts are made, as well as the config for the status bar, we're almost done! Just one more thing...

Wrapping up: installing a Pocket C.H.I.P. oriented general config for i3

Now to replace the standard i3wm config with a custom one with a mode to change settings, as well as shutdown, and sleep functions, start by changing directory to your ~/.i3 folder.

cd ~/.i3

Now, since we already generated a config file, we'll need to be replacing it. There's a few ways this can be done, but creating a new file and renaming it to "config" is probably the most straightforward. Create a new text file just named "config1", for simplicity's sake.

sudo nano config1

Now, you're free to copy this entire chunk and paste it into config1:

==# This file has been auto-generated by i3-config-wizard(1).
# It will not be overwritten, so edit it as you like.
#
# Should you change your keyboard layout some time, delete
# this file and re-run i3-config-wizard(1).
#
    
# i3 config file (v4)
#
# Please see http://i3wm.org/docs/userguide.html for a complete reference!

set $mod Mod1

# Commands to be run automatically on startup
exec --no-startup-id "xmodmap ~/.Xmodmap"

# Font for window titles. Will also be used by the bar unless a different font
# is used in the bar {} block below.
# This font is widely installed, provides lots of unicode glyphs, right-to-left
# text rendering and scalability on retina/hidpi displays (thanks to pango).
font pango:DejaVu Sans Mono 8
# Before i3 v4.8, we used to recommend this one as the default:
# font -misc-fixed-medium-r-normal--13-120-75-75-C-70-iso10646-1
# The font above is very space-efficient, that is, it looks good, sharp and
# clear in small sizes. However, its unicode glyph coverage is limited, the old
# X core fonts rendering does not support right-to-left and this being a bitmap
# font, it doesn't scale on retina/hidpi displays.

# Use Mouse+$mod to drag floating windows to their wanted position
floating_modifier $mod

# start a terminal
bindsym $mod+Return exec terminator

bindsym $mod+Escape exec sudo shutdown now

# kill focused window
bindsym $mod+q kill

# start dmenu (a program launcher)
# bindsym $mod+d exec dmenu_run
# There also is the (new) i3-dmenu-desktop which only displays applications
# shipping a .desktop file. It is a wrapper around dmenu, so you need that
# installed.
bindsym $mod+d exec --no-startup-id i3-dmenu-desktop

# change focus
bindsym $mod+j focus left
bindsym $mod+k focus down
bindsym $mod+l focus up
bindsym $mod+semicolon focus right
        
# alternatively, you can use the cursor keys:
bindsym $mod+Left focus left
bindsym $mod+Down focus down
bindsym $mod+Up focus up
bindsym $mod+Right focus right

# move focused window
bindsym $mod+Shift+j move left
bindsym $mod+Shift+k move down
bindsym $mod+Shift+l move up
bindsym $mod+Shift+colon move right

# alternatively, you can use the cursor keys:
bindsym $mod+Shift+Left move left
bindsym $mod+Shift+Down move down
bindsym $mod+Shift+Up move up
bindsym $mod+Shift+Right move right

# split in horizontal orientation
bindsym $mod+h split h

# split in vertical orientation
bindsym $mod+v split v

# enter fullscreen mode for the focused container
bindsym $mod+f fullscreen

# change container layout (stacked, tabbed, toggle split)
bindsym $mod+s layout stacking
bindsym $mod+w layout tabbed
bindsym $mod+e layout toggle split

# toggle tiling / floating
bindsym $mod+Shift+space floating toggle

# change focus between tiling / floating windows
bindsym $mod+space focus mode_toggle

# focus the parent container
bindsym $mod+a focus parent

# focus the child container
#bindsym $mod+d focus child

# switch to workspace
bindsym $mod+1 workspace 1
bindsym $mod+2 workspace 2
bindsym $mod+3 workspace 3
bindsym $mod+4 workspace 4
bindsym $mod+5 workspace 5
bindsym $mod+6 workspace 6
bindsym $mod+7 workspace 7
bindsym $mod+8 workspace 8
bindsym $mod+9 workspace 9
bindsym $mod+0 workspace 10

# move focused container to workspace
bindsym $mod+Shift+1 move container to workspace 1
bindsym $mod+Shift+2 move container to workspace 2
bindsym $mod+Shift+3 move container to workspace 3
bindsym $mod+Shift+4 move container to workspace 4
bindsym $mod+Shift+5 move container to workspace 5
bindsym $mod+Shift+6 move container to workspace 6
bindsym $mod+Shift+7 move container to workspace 7
bindsym $mod+Shift+8 move container to workspace 8
bindsym $mod+Shift+9 move container to workspace 9
bindsym $mod+Shift+0 move container to workspace 10

# reload the configuration file
bindsym $mod+Shift+c reload
# restart i3 inplace (preserves your layout/session, can be used to upgrade i3)
bindsym $mod+Shift+r restart
# exit i3 (logs you out of your X session)
bindsym $mod+Shift+e exec "i3-nagbar -t warning -m 'You pressed the exit shortcut. Do you really want to exit i3? This will end your X session.' -b 'Yes, exit i3' 'i3-msg exit'"

# resize window (you can also use the mouse for that)
mode "resize" {
        # These bindings trigger as soon as you enter the resize mode

        # Pressing left will shrink the window's width.
        # Pressing right will grow the window's width.
        # Pressing up will shrink the window's height.
        # Pressing down will grow the window's height.
        bindsym j resize shrink width 10 px or 10 ppt
        bindsym k resize grow height 10 px or 10 ppt
        bindsym l resize shrink height 10 px or 10 ppt
        bindsym semicolon resize grow width 10 px or 10 ppt

        # same bindings, but for the arrow keys
        bindsym Left resize shrink width 10 px or 10 ppt
        bindsym Down resize grow height 10 px or 10 ppt
        bindsym Up resize shrink height 10 px or 10 ppt
        bindsym Right resize grow width 10 px or 10 ppt

        bindsym $mod+r mode "default"
}

mode "config" {
        # Pressing left lowers the volume
        # Pressing right raises volume
        # Pressing up raises the brightness
        # Pressing down lowers the brightness
        bindsym Left exec amixer set "Power Amplifier" 5%-
        bindsym Right exec amixer set "Power Amplifier" 5%+
        bindsym Up exec brightness up
        bindsym Down exec brightness down

        bindsym j exec amixer set "Power Amplifier" 5%-
        bindsym k exec amixer set "Power Amplifier" 5%+
        bindsym l exec brightness up
        bindsym semicolon exec brightness down

        bindsym $mod+p mode "default"
}

mode "suspended" {
        bindsym $mod+BackSpace exec nmcli radio wifi on && sleep 1; mode "default"
}


bindsym $mod+p mode "config"

bindsym $mod+r mode "resize"

bindsym $mod+BackSpace exec nmcli radio wifi off && sleep 1 && xset dpms force off; mode "suspended"

# Start i3bar to display a workspace bar (plus the system information i3status
# finds out, if available)
bar {
        status_command i3blocks
}

for_window [class="^.*"] border pixel 1
new_window 1pixel
This config file creates a couple super handy new features. One is the mod+R resize mode, in which you can change how large or small you want the current focused window to be with the arrow keys. Another is the mod+P settings mode, in which the up and down arrows can be used to change brightness, and left and right can be used to change volume. Finally, there's a sort of pseudo-mode toggled via mod+BackSpace which puts the Pocket C.H.I.P. into a screen off, networking off, low energy consumption mode.

Switch to this new config using

sudo mv config1 config

then hit mod+Shift+R to reload i3 with the custom configurations we've done under effect. Done!
