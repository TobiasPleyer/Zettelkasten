---
date:  2025-07-01
tags:
---

# Thinkpad fan control

https://blog.monosoul.dev/2021/10/17/how-to-control-thinkpad-p14s-fan-speed-in-linux/

Recently I got a new laptop for work – Lenovo ThinkPad P14s (Gen 2). I prefer
running Linux on my working machines, and this one is not an exclusion. While
using it I noticed that the fan speed is sometimes too high for no reason
creating undesirable noise. Here’s how to control ThinkPad P14s’ fan speed in
Linux using a nice tool – thinkfan.

The steps described below are valid for Ubuntu 21.10, but probably shouldn’t be
much different for any other distrib.

Contents hide
1 TL;DR
2 Configuration deep dive
2.1 Sensors section
2.2 Fans section
2.3 Levels section
3 Check fan speeds available
4 Bonus
4.1 thinkfan-sleep.service
4.2 thinkfan-wakeup.service
4.3 Apply the changes
5 Summary

TL;DR

Time needed: 5 minutes

Here’s how to set everything up

 1. Enable fan control

    echo 'options thinkpad_acpi fan_control=1' | sudo tee /lib/modprobe.d/
    thinkpad_acpi.conf

 2. Install thinkfan package

    sudo apt install thinkfan

 3. Create a new thinkfan configuration file

    sudo touch /etc/thinkfan.conf

 4. Put the following lines into /etc/thinkfan.conf

    sensors:
      # GPU
      - tpacpi: /proc/acpi/ibm/thermal
        indices: [1]
      # CPU
      - hwmon: /sys/class/hwmon
        name: coretemp
        indices: [2, 3, 4, 5]
      # Chassis
      - hwmon: /sys/class/hwmon
        name: thinkpad
        indices: [3, 5, 6, 7]
      # SSD
      - hwmon: /sys/class/hwmon
        name: nvme
        indices: [1, 2, 3]
        correction: [-5, 0, 0]
      # MB
      - hwmon: /sys/class/hwmon
        name: acpitz
        indices: [1]

    fans:
      - tpacpi: /proc/acpi/ibm/fan

    levels:
      - [0, 0, 37]
      - [1, 35, 42]
      - [2, 40, 45]
      - [3, 43, 47]
      - [4, 45, 52]
      - [5, 50, 57]
      - [6, 55, 72]
      - [7, 70, 82]
      - ["level full-speed", 77, 32767]

 5. Configure thinkfan to use the newly created file

    echo 'THINKFAN_ARGS="-c /etc/thinkfan.conf"' | sudo tee -a /etc/default/
    thinkfan

 6. Enable thinkfan service

    sudo systemctl enable thinkfan

 7. Reboot

    sudo reboot

With this configuration the fan will be kept at a reasonably quiet level while
still allowing you to use the ThinkPad on top of your laps. 🙂

Configuration deep dive

If you’re interested in what exactly all the above-mentioned configuration
options do, welcome to this deep dive section.

Sensors section

Sensors section of the configuration file describes all the thermal sensors
thinkfan will use to keep an eye on the temperature.

‼️ IMPORTANT‼️: avoid using nvml option (temperatures read from proprietary
nVidia GPU driver).

Using nvml prevents the GPU from switching to suspend power state, causing
battery drain and high temperatures.

Here’s what we have there:

  * hwmon – specifies a generic temperature sensor. We use it to monitor CPU,
    SSD, WLAN and other devices’ temperature. A you can see, we have multiple
    hwmon entries having different names in the configuration file, here’s what
    they are:
      + tpacpi (#GPU) – this array of sensors represents pretty much the same
        temperatures as the one below (the chassis one), but with a little
        difference: the sensors are never missing here even if the device is
        offline (like GPU for example). Instead such sensors show negative
        values here. And this is what we’re using the sensor with index 1 for –
        to measure the GPU temperature.
        Technically, thinkfan should support devices that can be removed/
        suspended, but there’s a bug preventing it from working properly in
        daemon mode.
      + coretemp (#CPU) – measures CPU temperature. My ThinkPad P14s has 4
        physical cores, therefore there are 5 sensors available: 1 sensor for
        the whole package + 1 sensor per core. Since there are sensors for
        every core, it feels redundant to use the package sensor as well,
        therefore we ignore it with this option:
        indices: [2, 3, 4, 5]
        Notice we skip sensor having index 1.
      + thinkpad (#Chassis) – measures CPU, GPU and chassis temperature. In my
        case sensor with index 8 is unavailable and sensor with index 4 always
        return 0, therefore I excluded them. Sensors 1 and 2 report CPU and GPU
        temperatures respectively, and since I already have other sensors to
        monitor CPU and GPU, I’ve excluded them as well. After all exclusions,
        this is what we have in the configuration:
        indices: [3, 5, 6, 7]
      + nvme – measures SSD temperature. There are 3 sensors available, where
        the first one is a composite sensor. I’ve noticed that during light
        internet browsing temperature of that sensor was typically higher than
        CPU temperature (I suppose because of browser using cache), while
        during heavy load CPU was hotter. Therefore I decided to correct this
        sensor’s temperature by -5 degrees to prevent the fan spinning up when
        it’s not really needed. So, we use all sensors available:
        indices: [1, 2, 3]
        And correct the temperature of the first sensor by -5 degrees:
        correction: [-5, 0, 0]
      + acpitz – measures motherboard temperature (or at least this is what I
        expect it to measure, not 100% sure about it). Has only 1 sensor that
        we use as is.

Fans section

ThinkPad P14s only has one fan, therefore not much to configure in that
section. It has only 1 entry:

tpacpi: /proc/acpi/ibm/fan

It specifies the path to the interface to control fan. By the way, make sure
you did the first step of TL;DR section in order to be able to control the fan
speed.

Levels section

In this section we configure the fan speed and temperature levels to enable
this fan speed. Thinkfan has 2 modes of fan speed configuration – simple and
detailed.

In simple mode thinkfan will use the highest temperature of all sensors to
determine fan speed, while with detailed mode you can configure fan speed on
per sensor basis.

I use the simple mode here, since ThinkPad P14s is a relatively small device
and one component heating up contributes to the heating of the whole device, so
using the highest temperature of all sensors work well.

Each entry of this section is an array of 3 values, where:

  * first item – is the fan speed level to use;
  * second item – is the temperature at which thinkfan should drop the fan
    speed to the previous level;
  * third item – is the temperature at which thinkfan should step up the fan
    speed to the next level.

Let’s take a few entries as an example to better understand what happens there:

- [1, 35, 42]
- [2, 40, 45]

When the temperature reaches 42°C, thinkfan will change the fan speed to level
2. Once the temperature drops down to 40 degrees thinkfan will change the fan
speed to level 1.

Having a bit of overlapping between ramp up and drop down make the speed
changes a bit smoother.

Check fan speeds available

To check what fan speeds you can use with your laptop, you can run the
following command:

cat /proc/acpi/ibm/fanCode language: Bash (bash)

The output should look somewhat like this:

status:         enabled
speed:          3370
level:          4
commands:       level <level> (<level> is 0-7, auto, disengaged, full-speed)
commands:       enable, disable
commands:       watchdog <timeout> (<timeout> is 0 (off), 1-120 (seconds))Code language: Gradle (gradle)

Bonus

If you’re also using ThinkPad P14s (Gen 2) with Intel CPU, you might have
noticed that it doesn’t support S3 sleep state. You can technically turn it on
in the BIOS, but I found it not working well. For example, the touchscreen
didn’t work after laptop wakes up. Therefore I decided to stick with the
default S2idle (used for Windows 10 and Linux by default).

Apart from power saving issues of that sleep state (we are not going to discuss
those in this article), there’s another rather annoying issue: the fan doesn’t
power off after you close the laptop lid and it enters the sleep state.

Luckily, if you already have thinkfan installed, it can help you with that
issue as well!

Apart from thinkfan.service systemd unit, thinkfan provides a few extra units:

  * thinkfan-sleep.service – executed when the system is going to sleep;
  * thinkfan-wakeup.service – executed when the system is waking up from sleep.

We’re going to use those units to disable the fan when the system goes to sleep
and to enable when it wakes up.

thinkfan-sleep.service

First, let’s change thinkfan-sleep.service.

sudo nano /lib/systemd/system/thinkfan-sleep.serviceCode language: Bash (bash)

You should put a new line at the end of Service block:
ExecStart=/usr/bin/bash -c '/usr/bin/echo disable > /proc/acpi/ibm/fan'

So the file would look like that:

[Unit]
Description=Notify thinkfan of imminent sleep
Before=sleep.target

[Service]
Type=oneshot
ExecStart=/usr/bin/pkill -x -winch thinkfan
# Hack: Since the signal handler races with the sleep, we need to delay a bit
ExecStart=/usr/bin/sleep 1
ExecStart=/usr/bin/bash -c '/usr/bin/echo disable > /proc/acpi/ibm/fan'

[Install]
WantedBy=sleep.targetCode language: Properties (properties)

thinkfan-wakeup.service

Now, let’s change thinkfan-wakeup.service.

sudo nano /lib/systemd/system/thinkfan-wakeup.serviceCode language: Bash (bash)

You should put a new line into Service block, right after Type=oneshot, but
before ExecStart=...:
ExecStart=/usr/bin/bash -c '/usr/bin/echo enable > /proc/acpi/ibm/fan'

So the file would look like that:

[Unit]
Description=Reload thinkfan after waking up from suspend
After=sysinit.target
After=suspend.target
After=suspend-then-hibernate.target
After=hybrid-sleep.target
After=hibernate.target

[Service]
Type=oneshot
ExecStart=/usr/bin/bash -c '/usr/bin/echo enable > /proc/acpi/ibm/fan'
ExecStart=/usr/bin/pkill -x -usr2 thinkfan

[Install]
WantedBy=sleep.targetCode language: Properties (properties)

Apply the changes

Now to apply the changes run:

sudo systemctl daemon-reloadCode language: Bash (bash)

And you’re good. Now whenever your laptop goes to sleep, even in s2idle state
the fan will be turned off.

Summary

Thanks to the flexibility of Linux we can control the fan speed however we
want, regardless of the factory setup.

I hope this wasn’t too hard and you got a better understanding of how to set
thinkfan up.

If you were able to come up with fan speed settings that you like more, please,
do let me know about your configs in the comment section below.

Thank you and happy hacking!

Like it? Share it!
          

  * linux
  * opensource

AUTHOR

Andrei Nevedomskii

Andrei Nevedomskii

I'm a software engineer with 10+ years of experience in IT. I'm passionate
about backend engineering, high load, testing, open source and hacking.
17 posts

You may also like

 Using CGroups to make local test runs less painful
 
Published September 15, 2020

Using CGroups to make local test runs less painful

In my team we’re using Docker a lot, especially in pair with testcontainers.
When we’re running functional tests locally, there might be spinned up
localstack, WireMock, Postgres and a container with the service itself. Running
them all at the same time as well as the tests might heat up your laptop a lot,
as well as make it next to unusable for quite some time. Luckily, thanks to
having Linux on my laptop, this issue can be solved using CGroups.

 Hello there!
Published September 14, 2020

Hello there!

 Automatically signing nVidia kernel module in Fedora 36
 
Published May 17, 2022

Automatically sign NVidia Kernel module in Fedora 38

67 comments

Here’s how to automatically sign NVIDIA Kernel module in Fedora to make it more
convenient to use with Secure Boot enabled.

 Automatically signing nVidia kernel module in Fedora 35
 
Published December 29, 2021

Automatically sign NVidia Kernel module in Fedora 35

18 comments

Here’s how to automatically sign NVIDIA Kernel module in Fedora 35 to make it
more convenient to use with Secure Boot enabled.

Leave a comment Cancel reply

Your email address will not be published. Required fields are marked *

         [                                             ]
         [                                             ]
         [                                             ]
         [                                             ]
         [                                             ]
         [                                             ]
         [                                             ]
Comment *[                                             ]

Name * [                              ]

Email * [                              ]

Website [                              ]

[ ] Save my name, email, and website in this browser for the next time I
comment.

[*]Notify me via e-mail if anyone answers my comment.

[ ]I consent to Monosoul's Dev Blog collecting and storing the data I submit in
this form. (Privacy Policy) *

[Post Comment] 

 [                                             ] 
 [                                             ] 
 [                                             ] 
 [                                             ] 
 [                                             ] 
 [                                             ] 
 [                                             ] 
Δ[                                             ] 

This site uses Akismet to reduce spam. Learn how your comment data is
processed.

31 thoughts on “How to control ThinkPad P14s’ fan speed in Linux”

  * 31&nbspcomments

  * [d948ba2aaf]

    kyle
    April 18, 2023, 08:06

    is there a way i could monitor the current state of thinkfan within a
    conky.rc file?

    Reply
  * [399a8b3313]

    Sergio
    September 19, 2022, 10:53

    Hi Andrei,
    in my Lenovo T15 Gen 2 – Type 20W4 – i had to set this indices on SSD
    section, as sensor 3 seems missing:

    “`
    # SSD
    – hwmon: /sys/class/hwmon
    name: nvme
    indices: [1, 2]
    correction: [-5, 0]
    “`

    Error was:
    “`
    set 19 09:38:12 sergio-ThinkPad-T15-Gen-2i systemd[1]: Starting Thinkfan,
    the minimalist fan control program…
    set 19 09:38:12 sergio-ThinkPad-T15-Gen-2i thinkfan[1166]: ERROR: /etc/
    thinkfan.conf:16:
    indices: [1, 2, 3]
    ^
    Unable to find requested temperature inputs in /sys/class/hwmon/hwmon3..
    set 19 09:38:12 sergio-ThinkPad-T15-Gen-2i systemd[1]: thinkfan.service:
    Control process exited, code=exited, status=1/FAILURE
    set 19 09:38:12 sergio-ThinkPad-T15-Gen-2i systemd[1]: thinkfan.service:
    Failed with result ‘exit-code’.
    set 19 09:38:12 sergio-ThinkPad-T15-Gen-2i systemd[1]: Failed to start
    Thinkfan, the minimalist fan control program.
    “`

    Thanks,
    Sergio

    Reply
      + Andrei Nevedomskii

        Andrei Nevedomskii Post author
        September 19, 2022, 10:58

        Hey Sergio,

        Thanks for sharing! Probably it depends on the SSD model.

        Reply
      + [c8cd266b04]

        larky
        May 22, 2024, 00:50

        This fixed it for me too! Legend!

        Reply
  * [28552446bb]

    BlaY0
    June 24, 2022, 14:41

    Hi Andrei,

    Just an observation… I also have P14s but Gen 2i (Intel). What I see is
    that in my case both Composite and Sensor 1 of hwmon3 (nvme) are approx. 5
    degrees higher than Sensor 2, around 50 degrees both. I’m also not quite
    sure about the impact that higher fan RPM would have on nvme module itself.
    I tried with full-speed for some time to see if temp would go down and it
    actually didn’t so I guess if I go with -10 for both Composite and Sensor 1
    and -5 for Sensor 2 it is good enough. I will do some more stress tests for
    nvme to see if fan speed has any impact on it at all. There might also be a
    hwmon problem (kernel bug)… wrongly reporting those temps?

    Reply
  * [217347f27d]

    Yiannis
    March 17, 2022, 23:07

    Hello Andrei,

    Thanks for sharing this to the rest of us. I just got the P14s G2 (intel) a
    few weeks ago. It doesn’t have the touch screen and after updating the
    firmware, I think I don’t have any issues with s3 state and having the fan
    always on. However, after following the first 7 steps of your guide,
    thinkfan is crashing.

    “`
    × thinkfan.service – Thinkfan, the minimalist fan control program
    Loaded: loaded (/usr/lib/systemd/system/thinkfan.service; enabled; vendor
    preset: disabled)
    Active: failed (Result: core-dump) since Thu 2022-03-17 23:59:25 EET; 6min
    ago
    Process: 7388 ExecStart=/usr/sbin/thinkfan -n $OPTIONS (code=dumped, signal
    =ABRT)
    Main PID: 7388 (code=dumped, signal=ABRT)
    CPU: 17ms

    Mar 17 23:59:25 gon systemd[1]: Started Thinkfan, the minimalist fan
    control program.
    Mar 17 23:59:25 gon systemd[1]: thinkfan.service: Main process exited, code
    =dumped, status=6/ABRT
    Mar 17 23:59:25 gon systemd[1]: thinkfan.service: Failed with result
    ‘core-dump’.
    “`

    “`
    sudo thinkfan -c /etc/thinkfan.conf
    /usr/include/c++/11/bits/stl_vector.h:1063: std::vector::const_reference
    std::vector::operator[](std::vector::size_type) const [with _Tp = int;
    _Alloc = std::allocator; std::vector::const_reference = const int&;
    std::vector::size_type = long unsigned int]: Assertion ‘__n size()’ failed.
    zsh: IOT instruction sudo thinkfan -c /etc/thinkfan.conf
    “`

    I am on Fedora 35

    Reply
      + Andrei Nevedomskii

        Andrei Nevedomskii Post author
        March 18, 2022, 01:01

        Hey Yiannis,
        I’m using Fedora 35 myself and I don’t use thinkfan with it, since on
        Fedora the fan is quiet for me out of the box

        Reply
          o [217347f27d]

            Yiannis
            March 18, 2022, 11:46

            What do you use for power management?
            The out of the box battery saver power-profile was throttling the
            cpu too far, and balanced turned the fan higher than I like. I
            installed cpufreq (+gnome extention) which allowed the cpu to turbo
            boost, and performance was ok even in power saving mode.

            I then uninstalled both of those and tried powertop –auto-tune with
            tlp. Now I don’t have any control over the power profile, but most
            of the time the laptop is quite and fast. But, sometimes, after
            finishing a demanding task, where the fan has to ramp up, it takes
            too long to slow back down, even when CPU temps are low.

            Do you have a recent article about your power management settings?

            Do you suggest I uninstall thinkfan since I am on Fedora? I thought
            it would be nice to have a little more control.

            Reply
              # Andrei Nevedomskii

                Andrei Nevedomskii Post author
                March 18, 2022, 12:21

                I don’t use anything specific, just default “balanced” profile.
                TBH, I don’t experience any performance issues with it, works
                great for me. At least when I’m on AC power. When it runs on
                battery, then yes, the laptop gets noticeably slower. But I’m
                fine with it since at least it’s quiet and cool enough to sit
                on my laps 🙂

                I’ve tried using TLP as well, but I think it was giving me some
                weird issues, I don’t remember exactly, but at some point I
                decided to uninstall it.

                The thing is Fedora 35 comes with the latest version of
                thermald and I think it gets the job done quite well. At least
                I’m pretty happy with what I get out of the box in comparison
                to how it was with Ubuntu.

                People have different use cases, so what works for me might not
                work for you. If you want to have more control over your fan,
                I’d say go for it. 🙂

                I’m not 100% sure what’s the issue you have with thinkfan, the
                error doesn’t tell much to me. Might be just a bug. One thing
                you can try – is to remove items from sensors list one by one
                and see if thinkfan works after each removal. Maybe it’s one of
                the sensors that makes it crash. A good candidate to start
                would be the GPU sensor definition.

                Also, since Fedora has a newer version of thinkfan, you can
                actually replace GPU sensor definition with this one:

                - hwmon: /sys/class/hwmon
                name: thinkpad
                indices: [ 2 ]
                optional: true

                It should work a bit better

                Reply
  * [7d2f2fbc79]

    George
    February 3, 2022, 17:14

    Hello, thanks for your guide, i want to ask you about this line:
    Configure thinkfan to use the newly created file
    echo ‘THINKFAN_ARGS=”-c /etc/thinkfan.conf”‘ | sudo tee -a /etc/default/
    thinkfan
    What is the point of that? I haven’t seen that anywhere, not even on the
    author’s github

    Reply
      + Andrei Nevedomskii

        Andrei Nevedomskii Post author
        February 3, 2022, 17:39

        Hey George,
        Thinkfan package in Ubuntu uses a different config location by default,
        this line is only needed in Ubuntu to make thinkfan use the newly
        created configuration.

        Reply
  * [6bac95326f]

    Christoph
    February 2, 2022, 15:12

    Hi Andrei,

    I’m on Bebian Bullseye 11.2 with a Lenovo Thinkpad P14s Gen2. Installed is
    thinkfan 0.9.1 (thinkfan-0.9.3-2).
    I pasted your /etc/thinkfan.conf:
    — snip —
    # 2022-02-02, 11:54:13, haasc:
    sensors:
    # GPU
    – tpacpi /proc/acpi/ibm/thermal
    indices: [1]
    # CPU
    – hwmon: /sys/class/hwmon
    name: coretemp
    indices: [2, 3, 4, 5]
    # Chassis
    – hwmon: /sys/class/hwmon
    name: thinkpad
    indices: [3, 5, 6, 7]
    # SSD
    – hwmon: /sys/class/hwmon
    name: nvme
    indices: [1, 2, 3]
    correction: [-5, 0, 0]
    # MB
    – hwmon: /sys/class/hwmon
    name: acpitz
    indices: [1]

    # fans:
    – tpacpi: /proc/acpi/ibm/fan

    levels:
    – [0, 0, 37]
    – [1, 35, 42]
    – [2, 40, 45]
    – [3, 43, 47]
    – [4, 45, 52]
    – [5, 50, 57]
    – [6, 55, 72]
    – [7, 70, 82]
    – [“level full-speed”, 77, 32767]
    — snap —
    but when I’m trying to start thinkfan, it fails with
    The unit thinkfan.service has entered the ‘failed’ state with result
    ‘exit-code’.
    thinkfan[28373]: /etc/thinkfan.conf:4:- tpacpi /proc/acpi/ibm/thermal
    thinkfan[28373]: Syntax error.
    thinkfan[28373]: Refusing to run without usable config file!
    systemd[1]: Failed to start simple and lightweight fan control program.
    ░░ Subject: A start job for unit thinkfan.service has failed
    ░░ Defined-By: systemd

    Your syntax for the thinkfan.conf seems quite different from what is in /
    usr/share/doc/thinkfan/examples.

    I have no clue, what is wrong and how it can be fixed. Can you point me in
    the right direction?
    Is maybe the version of thinkfan shipped with Debian Bullseye too old? Do I
    have to install v1.2 from Debian Testing?

    Thanks
    Christoph.

    Reply
      + Andrei Nevedomskii

        Andrei Nevedomskii Post author
        February 2, 2022, 17:10

        Hey Christoph,

        Thinkfan supports 2 syntaxes: flat and structured. The syntax I use
        here is YAML (the structured one), I think it was introduced to
        thinkfan at version 1.0, so yeah, probably your version is too old. If
        you can, I’d recommend installing version 1.2 from Debian testing since
        it has quite a few bugfixes. Also, I it should work with the config
        example from here after you do that.

        Although, if you’re not very fond of Debian, I can recommend installing
        Fedora 35 on your laptop. I’m running it right now on my P14s and it
        works great, I don’t even use thinkfan anymore, since the fan is quiet
        out of the box.

        Reply
  * [b3a2021944]

    druizz
    December 15, 2021, 11:13

    Hi Andrei, thank you for the tutorial.

    I am using Arch and I configured `thinkfan` with your guide, but I have
    problems.

    The service is running OK, and it is able to change the fan configuration
    with my configuration. The problem is that although the configuration is
    set, the fan is always active. This is the output of the `thinkfan`
    service:

    “`
    systemctl status thinkfan
    ● thinkfan.service – simple and lightweight fan control program
    Loaded: loaded (/usr/lib/systemd/system/thinkfan.service; enabled; vendor
    preset: disabled)
    Drop-In: /etc/systemd/system/thinkfan.service.d
    └─override.conf
    Active: active (running) since Wed 2021-12-15 11:04:50 CET; 5min ago
    Process: 14552 ExecStart=/usr/bin/thinkfan $THINKFAN_ARGS (code=exited,
    status=0/SUCCESS)
    Main PID: 14553 (thinkfan)
    Tasks: 1 (limit: 18785)
    Memory: 532.0K
    CPU: 24ms
    CGroup: /system.slice/thinkfan.service
    └─14553 /usr/bin/thinkfan -b0

    Dec 15 11:04:50 druizz-pc systemd[1]: Starting simple and lightweight fan
    control program…
    Dec 15 11:04:50 druizz-pc thinkfan[14552]: Daemon PID: 14553
    Dec 15 11:04:50 druizz-pc systemd[1]: Started simple and lightweight fan
    control program.
    Dec 15 11:04:50 druizz-pc thinkfan[14553]: Temperatures(bias): 40(0), 43
    (0), 34(0), 37(0) -> Fans: level 0
    Dec 15 11:10:05 druizz-pc thinkfan[14553]: Temperatures(bias): 38(0), 56
    (0), 35(0), 85(0) -> Fans: level full-speed
    Dec 15 11:10:07 druizz-pc thinkfan[14553]: Temperatures(bias): 34(0), 37
    (0), 32(0), 33(0) -> Fans: level 0
    “`

    As you can see, the level changes with the temperatures, but my fan still
    runs.

    If I stop the `thinkfan` service and set the configuration manually, I can
    see the change when I execute `cat /proc/acpi/ibm/fan`, but the fan still
    runs although the level 0 is set:

    “`

    Reply
      + [b3a2021944]

        druizz
        December 15, 2021, 11:25

        Sorry, I pressed the enter key by error 🙂

        As I wrote above, `thinkfan` is able to edit `/proc/acpi/ibm/fan`, but
        the changes seem to not be applied. The fan speed always shows `32768`,
        it would be 0 or 7:

        “`
        % cat /proc/acpi/ibm/fan
        status: disabled
        speed: 32768
        level: 0
        commands: level ( is 0-7, auto, disengaged, full-speed)
        commands: enable, disable
        commands: watchdog ( is 0 (off), 1-120 (seconds))
        “`

        After setting level 7 manually:

        “`
        cat /proc/acpi/ibm/fan
        status: enabled
        speed: 32768
        level: 7
        commands: level ( is 0-7, auto, disengaged, full-speed)
        commands: enable, disable
        commands: watchdog ( is 0 (off), 1-120 (seconds))
        “`

        This is my configuration, I only monitor the CPU temperature:

        “`
        % cat /etc/thinkfan.conf
        sensors:
        – hwmon: /sys/class/hwmon
        name: coretemp
        indices: [2, 3, 4, 5]
        fans:
        – tpacpi: /proc/acpi/ibm/fan
        levels:
        – [0, 0, 60]
        – [2, 60, 65]
        – [3, 65, 70]
        – [5, 70, 75]
        – [6, 75, 80]
        – [7, 80, 85]
        – [“level full-speed”, 85, 32767]
        “`
        About my model, is a Thinkpad L13 2 Gen.

        Have you any idea for solving the problem? I just updated the BIOS to
        the last version, but the fan still runs in level 0.

        Thank you!

        Reply
          o Andrei Nevedomskii

            Andrei Nevedomskii Post author
            December 15, 2021, 11:34

            Hey druizz,
            Hmm, interesting, from the output you provided it seems like the
            fan runs at the same speed no matter what level you have specified
            there.

            Can you install lm-sensors package (at least that’s the package
            name in Ubuntu, not sure if it’s the same in Arch), then run
            sensors command and provide it’s output here?

            Reply
              # [12a6d62ba7]

                druizz
                December 15, 2021, 13:37

                Thank you for your quick response!

                I have installed this package, this is the output:

                sensors
                ucsi_source_psy_USBC000:002-isa-0000
                Adapter: ISA adapter
                in0: 5.00 V (min = +5.00 V, max = +5.00 V)
                curr1: 3.00 A (max = +3.00 A)

                ucsi_source_psy_USBC000:001-isa-0000
                Adapter: ISA adapter
                in0: 0.00 V (min = +0.00 V, max = +0.00 V)
                curr1: 0.00 A (max = +0.00 A)

                thinkpad-isa-0000
                Adapter: ISA adapter
                fan1: 32768 RPM
                temp1: +1.0°C
                temp2: +1.0°C
                temp3: +4.0°C
                temp4: +65.0°C
                temp5: +60.0°C
                temp6: +60.0°C
                temp7: +16.0°C
                temp8: +66.0°C
                temp9: +64.0°C
                temp10: +1.0°C
                temp11: -80.0°C
                temp12: +0.0°C
                temp13: +0.0°C
                temp14: +64.0°C
                temp15: +0.0°C
                temp16: +0.0°C

                BAT0-acpi-0
                Adapter: ACPI interface
                in0: 17.40 V

                iwlwifi_1-virtual-0
                Adapter: Virtual device
                temp1: +39.0°C

                coretemp-isa-0000
                Adapter: ISA adapter
                Package id 0: +62.0°C (high = +100.0°C, crit = +100.0°C)
                Core 0: +62.0°C (high = +100.0°C, crit = +100.0°C)
                Core 1: +61.0°C (high = +100.0°C, crit = +100.0°C)
                Core 2: +57.0°C (high = +100.0°C, crit = +100.0°C)
                Core 3: +61.0°C (high = +100.0°C, crit = +100.0°C)

                nvme-pci-0400
                Adapter: PCI adapter
                Composite: +39.9°C (low = -20.1°C, high = +77.8°C)
                (crit = +81.8°C)
                Sensor 1: +39.9°C (low = -273.1°C, high = +65261.8°C)

                acpitz-acpi-0
                Adapter: ACPI interface
                temp1: +63.0°C (crit = +99.0°C)

                Reply
                  @ Andrei Nevedomskii

                    Andrei Nevedomskii Post author
                    December 15, 2021, 14:45

                    Hmm, it looks like your fan is always at max rpm.
                    I’ve recently got a firmware update from Lenovo for my
                    laptop that has broken fan control for me as well, by
                    default the fan was always at 0 RPM in Linux no matter the
                    temperature. But they fixed that with another update a week
                    later, so it works fine now. Also, manual fan control (via
                    the proc file) was working fine regardless of the update.
                    Knowing how chaotic Arch could be sometimes, I’d try
                    spinning up the latest Ubuntu from a USB drive and see if
                    the `sensors` output is the same there. If it will be the
                    same, then it might be that you have a faulty fan and might
                    need to contact Lenovo’s warranty support.

                    Reply
                      - [b3a2021944]

                        druizz
                        December 16, 2021, 08:36

                        Yes, I’ll try with the same version of Ubuntu than you-

                        Yesterday I updated my BIOS with the last version, but
                        nothing changes 🙁

                        I also installed other kernel in Arch (I use the LTS so
                        I installed the “normal” that is more recent) and the
                        results are the same.

                        Thank you for your support, I’ll write an update with
                        the result of the test.

                        Reply
                          = Andrei Nevedomskii

                            Andrei Nevedomskii Post author
                            December 16, 2021, 08:50

                            One more thing you can try to do is follow this
                            guide: https://forum.manjaro.org/t/
                            how-to-choose-the-proper-acpi-kernel-argument/1405.
                            To be more specific, I think you can try acpi_osi=
                            'Windows 2017' (i.e. tell your BIOS you’re running
                            Windows 10), or maybe try apci_osi=Linux instead.
                            AFAIK Linux kernel pretends to be Windows by
                            default, since that provides the best
                            compatibility. So maybe setting acpi_osi to Linux
                            will change how the fan works for you. Give it a
                            shot!

                            Reply
                              x [12a6d62ba7]

                                druizz
                                December 16, 2021, 12:00

                                Just tested with `Windows 2017` and `Linux`
                                with both kernels (LTS and normal) without
                                success 🙁 Thank you for giving me support
                                again! It’s really weird

                                Reply
                                  % Andrei Nevedomskii

                                    Andrei Nevedomskii Post author
                                    December 16, 2021, 12:22

                                    Sorry to hear that. I hope you get it
                                    solved one way or another! My bet is that
                                    it’s a hardware issue.

                                    Reply
                                      * [b3a2021944]

                                        druizz
                                        December 21, 2021, 15:13

                                        With the latest version of Ubuntu, I
                                        was not able to start in normal mode
                                        (black screen), and after tried with
                                        the safe mode, a kernel panic happened
                                        and my bootloader was broken (I had to
                                        reinstall GRUB with a rescue USB image,
                                        very strage).

                                        I am going to ask in Lenovo forums, and
                                        probably I will send the laptop to the
                                        technical service.

                                        Thank you for your support, and for
                                        your guide, I will use it again when my
                                        laptop would be repaired (or replaced).

                      - [b3a2021944]

                        druizz
                        December 16, 2021, 10:37

                        For adding more detail, this is my configuration of the
                        `thinkpad_acpi` module:

                        sudo systool -m thinkpad_acpi -av
                        Module = “thinkpad_acpi”

                        Attributes:
                        coresize = “118784”
                        initsize = “0”
                        initstate = “live”
                        refcnt = “0”
                        srcversion = “BC9414C14C416F6770115BF”
                        taint = “”
                        uevent =
                        version = “0.26”

                        Parameters:
                        brightness_enable = “2”
                        brightness_mode = “4”
                        enable = “Y”
                        experimental = “0”
                        fan_control = “Y”
                        force_load = “N”
                        id = “ThinkPadEC”
                        index = “-536870912”
                        software_mute = “Y”
                        volume_capabilities = “0”
                        volume_control = “N”
                        volume_mode = “3”

                        Reply
  * [278f945584]

    Gordan Ovčarić
    December 11, 2021, 19:44

    Hello!
    First and foremost, thank you for this tutorial! :)))

    Second, my question:
    I’m trying to set-up thinkfan for my Lenovo P1 Gen4 (because it is
    ridiculous to me that fan is 2500rpm on CPU temp 45C).
    And the command ‘cat /proc/acpi/ibm/fan’ returns only:
    status: enabled
    speed: 2469
    level: auto

    How do I know how to set-up the levels then? How do I know how many levels
    are there?

    Thanks! 🙂

    P.S. is this tutorial “permanent” solution -> do it once and forget about
    it, or will it reset after pc is turned off/restarted?

    Reply
      + Andrei Nevedomskii

        Andrei Nevedomskii Post author
        December 11, 2021, 21:09

        Hey Gordan, you’re welcome!

        Regarding your question: is the output you shared are the only 3 lines
        you get when you call cat /proc/acpi/ibm/fan? There are supposed to be
        3 extra lines starting with commands (see the example output in the
        article), what you need is the line like this level ( is 0-7, auto,
        disengaged, full-speed). Basically, what it says is: there are 8 levels
        available – 0 to 7 (+ extra such as: auto, disengaged and full-speed).

        And yeah, the solution is permanent, so you configure it once and
        forget about it.

        Reply
          o [278f945584]

            Gordan Ovčarić
            December 11, 2021, 23:19

            Hey, Andrei!

            Thank you for your reply! 🙂

            Yes, those three lines are all I get from mentioned command (I’ve
            tried also locating the file, nothing more is inside when I look at
            it with Text Editor).
            For example, in that folder I found “driver” file and it says
            “ThinkPad ACPI Extras + version 0.26”, if that helps.
            Also, “bluetooth” file in that folder has ‘commands: enable,
            disable’ line….

            First question: Is it safe to assume that I also have this 0-7
            levels as you do?
            Second question: If I want to revert, and undo this setup, how to
            do it?

            Thank you a lot for your help! 🙂

            Reply
              # Andrei Nevedomskii

                Andrei Nevedomskii Post author
                December 12, 2021, 13:15

                Hey Gordan,
                Ah, ok, I see. Probably you don’t see the “commands” lines
                because you haven’t enabled fan control. It’s the first item in
                the article’s list of how to set it up.
                Unless you see the “commands” lines, there’s no way to control
                the fan speed, i.e. you can’t send commands to control the fan
                speed.
                To answer your first question: I wouldn’t just assume that,
                it’s better to be sure.
                As for the second question: it’s rather easy to undo it, just
                remove thinkfan app with `sudo apt remove thinkfan`.

                Reply
                  @ [278f945584]

                    Gordan Ovčarić
                    December 12, 2021, 22:45

                    Hey Andrei,

                    Thanks a lot for your answers, it is clear to me now 🙂

                    Yeah, I didn’t start the whole process until I was sure
                    what I was doing. Now, after your explanations, I
                    understand it all and will try it 🙂

                    Thanks again, and all the best 🙂

                    Reply
                      - Andrei Nevedomskii

                        Andrei Nevedomskii Post author
                        December 13, 2021, 00:11

                        Hey Gordan,
                        You’re welcome!
                        If you end up with a different configuration of
                        temperatures/fan speeds, please, share it in the
                        comments here.
                        I’d be curious to try it myself and other people might
                        find it useful!

                        Reply
                  @ [28552446bb]

                    BlaY0
                    June 24, 2022, 13:24

                    Hi Andrei,

                    Maybe it is worth noting that when you add fan_control
                    option to the thinkpad_acpi module in /etc/modprobe.d you
                    probably have to rebuild initramfs since module could
                    already be loaded at that stage. To see whether option was
                    really applied, one can check /sys/module/thinkpad_acpi/
                    parameters/fan_control (contains Y if enabled or N if not).

                    Reply

Post navigation

  * Previous post Ubuntu 21.10 and acpi-call-dkms bug
  * Back to post list
  * Next post How to unblock yourself

Tags

agile (1) containers (1) database (2) developer (2) development (9) docker (3)
engineering (1) fedora (1) gradle (1) jacoco (1) java (6) jooq (1) kotlin (7)
kubernetes (1) lean startup (1) linux (7) liquibase (1) management (1) metrics
(1) monitoring (1) opensource (7) ORM (1) software (6) spring (5) spring boot
(1) spring framework (5) ubuntu (1)

Categories

  * development (11)
  * linux (6)
  * management (1)
  * offtopic (6)

Meta

  * Log in
  * Entries feed
  * Comments feed
  * WordPress.org

Privacy policy

Hosted in Germany

© 2025 Monosoul's Dev Blog – All rights reserved

  *  
  *  
  *  
  *  

