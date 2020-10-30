I love the [matrix](https://matrix.org/) protocol and I've been migrating all my chats to it over
the past few months.

one pain point of matrix for me has been the lack of good native clients. I absolutely hate using
the element web app, it's sluggish and clunky and isn't very keyboard driven.

having to deal with encryption safely and properly has also been another hurdle for client
developers.

fortunately, I have recently found a good solution for both of these issues.

[pantalaimon](https://github.com/matrix-org/pantalaimon) is a daemon that acts as a middleman
matrix homeserver and deals with encryption for you. being a mitm server, it also lets you lock
down the security further, for example by default it will detect if a room has any unverified
sessions and ask you to manually confirm the messages before they go through.

the concept of pantalaimon is extremely powerful. now client developers can just implement a simple
unencrypted client and point it to pantalaimon and get access to encrypted rooms for free. this
minimizes lines of code and code duplication across clients, as well as potentially unsafe
implementations.

pantalaimon can also deal with verification for you, so your client doesn't need to do it.

so, how did pantalaimon enable me to ditch element? well, there's a nice experimental client named
QuickMedia which is a consistent keyboard driven interface to many other services such as youtube,
4chan and various manga sites. it's a native X11 application written in c++, it's extremely fast
and lightweight, it opens videos in mpv embedded in the window just how I want it and the developer
recently added matrix support. it's the ideal matrix client and pantalaimon enables it to work
on encrypted rooms.

here's a quick demo of how fast it is: https://streamable.com/mu9xvf

![](pics/quickmedia.gif)

# setting up pantalaimon
install pantalaimon. if you're on an arch-based linux distro it's available on the aur.
for other distros, you can get it from pip as explained in the [README](https://github.com/matrix-org/pantalaimon#installation)

create the pantalaimon config dir with `mkdir ~/.config/pantalaimon`

now, edit `~/.config/pantalaimon/pantalaimon.conf`

this is a sample config that hosts the proxy homeserver at `http://localhost:8009` and connects
to the homeserver `https://matrix.animegirls.xyz:8448` which is my private homeserver.
if you wanted to use the public midov.pl homeserver, for example, you would use
`https://midov.pl:8448` .

my config also suppresses manual verification for encrypted rooms with unverified sessions.
remove the `IgnoreVerification` line if you don't want that.

    [local-matrix]
    Homeserver = https://matrix.animegirls.xyz:8448
    ListenAddress = localhost
    ListenPort = 8009
    IgnoreVerification = True

now you are ready to start pantalaimon. on arch linux, the aur package for pantalaimon provides
a systemd service to autostart it which you can enable with `sudo systemctl enable pantalaimon`
and start with `sudo systemctl start pantalaimon` .

personally, I use runit on artix linux, so I created my custom runit service.

first of all, I have a runsvdir service to configure services that run as my user in my home:

    mkdir -p ~/.config/sv ~/.config/service
    sudo mkdir /etc/runit/sv/runsvdir-loli

    sudo cat > /etc/runit/sv/runsvdir-loli/run << "EOF"
    #!/bin/sh
    exec 2>&1
    exec chpst -uloli runsvdir /home/loli/.config/service
    EOF

    sudo chmod +x /etc/runit/sv/runsvdir-loli/run
    sudo ln -s /etc/runit/sv/runsvdir-loli/ /run/runit/service
    sudo sv start runsvdir-loli

then I have this service that starts pantalaimon and outputs the logs to `~/pantalaimonlog`

    mkdir ~/.config/sv/pantalaimon

    cat > ~/.config/sv/pantalaimon/run << "EOF"
    #!/bin/sh
    exec env HOME=/home/loli DISPLAY=:0 pantalaimon --log-level debug 2>&1 | tee ~/pantalaimonlog
    EOF

    chmod +x ~/.config/sv/pantalaimon/run
    ln -s ../sv/pantalaimon ~/.config/service
    sv start pantalaimon

now pantalaimon should be running, check the log file to see if it started up correctly

you can run `panctl` and check out the commands if you want, but with this config you will almost
never need to use that except for verification

# setting up QuickMedia
if you are on arch-based distros you can install quickmedia-git on the AUR, or get binary builds
from [chaotic-aur](https://lonewolf.pedrohlc.com/chaotic-aur/) . if you are on other distros,
below is a guide to compile it from source.

now, run `QuickMedia matrix` or create a shortcut that calls it and enter your user/password, but
set the homeserver to `http://localhost:8009` . it should connect and you should have access to
encrypted rooms. verify your pantalaimon session from element so it can decrypt messages by opening
the user list for a room you are in, clicking on your user and approving the session in the list
of sessions.

check out the [about page](https://git.dec05eba.com/QuickMedia/about/) of QuickMedia to learn
the keyboard shortcuts

# compiling QuickMedia from source
if your distro doesn't package QuickMedia, here's how to build from source

first of all install git

now run these commands:

    git clone --recursive https://git.dec05eba.com/sibs
    cd sibs
    ./cmake/install.sh
    cd ..
    git clone --recursive https://git.dec05eba.com/QuickMedia
    cd QuickMedia
    sibs build --release
    sudo mkdir /usr/share/quickmedia/
    sudo cp -r images/ icons/ shaders/ boards.json input.conf /usr/share/quickmedia/
    sudo cp launcher/* /usr/share/applications/
    sudo cp "sibs-build/$(sibs platform)/release/QuickMedia" /usr/bin/

now you can continue from "setting up QuickMedia"

to update, you can do this (replace /path/to with where those dirs are located)

    cd /path/to/sibs
    git fetch --all --recurse-submodules=yes
    git reset --hard origin/HEAD
    git clean -dffx
    git submodule update --recursive
    ./cmake/install.sh

    cd /path/to/QuickMedia
    git fetch --all --recurse-submodules=yes
    git reset --hard origin/HEAD
    git clean -dffx
    git submodule update --recursive
    sibs build --release
    sudo mkdir /usr/share/quickmedia/
    sudo cp -r images/ icons/ shaders/ boards.json input.conf /usr/share/quickmedia/
    sudo cp launcher/* /usr/share/applications/
    sudo cp "sibs-build/$(sibs platform)/release/QuickMedia" /usr/bin/

put these commands in a script for quick usage
