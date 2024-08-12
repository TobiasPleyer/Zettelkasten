---
date:  Sunday, March 24, 2024
tags:
---

# How To Install Home Assistant on Proxmox: The Easy Way

[Original](https://smarthomescene.com/guides/how-to-install-home-assistant-on-proxmox-the-easy-way/)

Skip to content
SmartHomeScene
 
Menu
Menu

  • Blog
  • News
  • Guides
  • Home Assistant
      □ Add-ons
      □ Integrations
      □ Cards
      □ Dashboards
      □ Proxmox
  • Device Reviews
      □ ZigBee
      □ Wi-Fi
      □ Bluetooth
      □ Thread
      □ Matter
  • DIY

 
SmartHomeScene
Menu

  • Blog
  • News
  • Guides
  • Home Assistant
      □ Add-ons
      □ Integrations
      □ Cards
      □ Dashboards
      □ Proxmox
  • Device Reviews
      □ ZigBee
      □ Wi-Fi
      □ Bluetooth
      □ Thread
      □ Matter
  • DIY

 

Be Smart, Go Local.

     
Guides
March 11, 2023

How To Install Home Assistant on Proxmox: The Easy Way

A guide showcasing the easiest way to install Home Assistant Operating System
on a Proxmox Virtual Environment, using a custom automated script.

Photo of authorPhoto of author
SHS
5 min
74 replies

Table of Contents

  • What's Proxmox?
  • Create a Proxmox Bootable USB Drive
  • Install Proxmox on your Mini PC
  • Installing Home Assistant on Promox
  • (Optional) USB Passthrough to Home Assistant VM
  • (Optional) Post Install Proxmox VE 7 Script
  • Summary

UPDATE 10.08.2023: You can now easily upgrade to Proxmox 8.0 without loosing
data, keeping all your VMs and containers intact. Follow the upgrade guide.

So, you’ve been using Home Assistant for a while now and decided to move from
the Raspberry Pi to something more powerful. You picked up a cheap used Intel
NUC Mini PC or maybe a refurbished Dell Optiplex and you want to deploy it as
your main Home Assistant server.

Easy Way to Setup Home Assistant on Proxmox VEEasy Way to Setup Home Assistant
on Proxmox VE

That’s all great, only you find out your NUC is really overpowered and would
like to have the option to run something else in parallel to your Home
Assistant instance. You’ve come across Promox a few times within the community,
but thought that it might be too complicated for you to setup and maintain.

Well, developer tteck created a bunch of helper scripts for deploying various
different VMs within Promox with simple commands, streamlining the setup
process significantly. We can use it to spin up a Home Assistant OS VM in
Proxmox and if needed, customize the dedicated resources the VM will use.

We can use these helper scripts to also setup a Zigbee2MQTT LXC and separate it
from Home Assistant, so it’s always up and running in case of Home Assistant
failure. Proper USB passthrough is also important for your Zigbee/Z-Wave
dongles or other peripherals you might want the VM to access.

If you already have Proxmox up and running, jump to Installing Home Assistant
on Proxmox section and continue from there.

What’s Proxmox?

Proxmox VE is a complete, open-source server management platform for enterprise
virtualization. It tightly integrates the KVM hypervisor and Linux Containers
(LXC), software-defined storage and networking functionality, on a single
platform. With the integrated web-based user interface you can manage VMs and
containers, high availability for clusters, or the integrated disaster recovery
tools with ease.

Proxmox Home Assistant ScreenshotProxmox Home Assistant ScreenshotSource:
proxmox.com

For the smart home use case, we can utilize Proxmox and it’s capabilities do
deploy different VMs and LXCs for Home Assistant, Zigbee2MQTT, ESPHome, MQTT
Broker or maybe Plex Servers and similar. This will allow you to really utilize
the mini PC you just bought.

Create a Proxmox Bootable USB Drive

Before installing Proxmox on your new Mini PC, first, you need to create a
bootable USB Drive with the latest version of Proxmox. You can do this by
flashing the ISO file either with Balena Etcher or Rufus.

1. Download Balena Etcher and install it
2. Download the latest Proxmox VE ISO Installer and save it
3. Insert the USB drive in your PC
4. Open Balena Etcher and click Flash from file

Proxmox Home Assistant Balena EtcherProxmox Home Assistant Balena Etcher

5. Select the Proxmox ISO file you just downloaded
6. Click Select target and choose your USB Drive

Proxmox Home Assistant Balena Etcher Choose TargetProxmox Home Assistant Balena
Etcher Choose Target

7. Click Flash and wait for the process to finish

8. Depending on the speed of your USB Drive, the process should take
~2-5minutes
9. You will get a Flash complete! confirmation

Proxmox Home Assistant Balena Etcher Flash CompleteProxmox Home Assistant
Balena Etcher Flash Complete

10. Close Balena Etcher
11. Safely remove hardware (USB)
12. You are ready to install Proxmox on your Mini PC

Install Proxmox on your Mini PC

1. Plug in the USB Drive you just created in your Mini PC
2. Power it on, and navigate to BIOS (Usually by pressing DEL or F10 key during
boot)
– Depending on your model, the BIOS will look differently and certain things
may be labelled differently. But in general, you need to tweak the following
settings:

– Make sure Secure Boot is disabled
– Make sure Legacy Boot is enabled
– Make sure Virtualization Technology is enabled

Proxmox Home Assistant Legacy BootProxmox Home Assistant Legacy Boot
Proxmox Home Assistant Legacy Virtualization TechnologyProxmox Home Assistant
Legacy Virtualization Technology

3. Save your BIOS settings and reboot
4. Enter your Boot Order Menu (Usually F8 key)
5. Select your USB Drive and hit enter
6. You will be presented with the Proxmox Welcome Screen
7. Select Install Proxmox VE

Proxmox Home Assistant Install Screen Step 1Proxmox Home Assistant Install
Screen Step 1

8. Accept the License Agreement and hit Next
9. Select the Hard Drive you want to install Proxmox to and hit Next
10. Select your Country, Time zone and Keyboard layout
– Make sure select the correct information, as Proxmox is heavily reliant on
the time zone to synchronize everything

Proxmox Home Assistant Install Screen Step 2 Location and Time ZoneProxmox Home
Assistant Install Screen Step 2 Location and Time Zone

11. Set a password and e-mail and click Next
– Password needs to be at least 8 characters, containing letters and numbers
– You will use this password for the initial Proxmox login
– E-mail is used for alert notifications (backup failures etc.)

Proxmox Home Assistant Install Screen Step 3 Login CredentialsProxmox Home
Assistant Install Screen Step 3 Login Credentials

12. Setup your Network configuration and click Next
– Set Hostname (pve.proxmox or pve.smarthomescene)
– IP Address, Gateway and DNS Settings should automatically populate if you
have the Mini PC plugged to your home network

Proxmox Home Assistant Install Screen Step 4 Network ConfigurationProxmox Home
Assistant Install Screen Step 4 Network Configuration

13. Wait for the process to finish
14. Installation successful should appear on the screen
15. Click Reboot and after a couple of minutes you can access your Promox
server from the IP_Address:8006 you gave it

Proxmox Home Assistant Install Screen Step 5 FinishedProxmox Home Assistant
Install Screen Step 5 Finished

Installing Home Assistant on Promox

1. Open your Proxmox instance by navigating to IP_Address:8006 from your main
PC
2. You will get a warning message Your connection is not private
3. Click Advanced and click Proceed to IP_Address

4. On the Promox login screen login with the credentials:
– Username: root
– Password: password you set during installation

Proxmox Home Assistant Install Login ScreenProxmox Home Assistant Install Login
Screen

5. You will get a message saying you do not have a valid subscription
– This is showing up because you don’t have a valid enterprise license
– We will clean this up with a script too
6. Before we deploy Home Assistant, we need to update Proxmox packages
7. On the left side of the screen, select your VM and click Updates
8. Click Refresh and click Upgrade

Proxmox Home Assistant Install Update PackagesProxmox Home Assistant Install
Update Packages

9. A dialog window will popup, going through the available package updates
10. You may get another license warning, just ignore it
11. If you get a confirmation dialog, type in “y” and hit enter
12. If you get a dialog asking the Keyboard encoding select UTF-8 and English
13. You will get a your system is up to date message

Proxmox Home Assistant Install Update Packages DoneProxmox Home Assistant
Install Update Packages Done

14. Close the window and you are done
15. If you click Refresh again, the package list should disappear since you
already updated them

16. To Install Home Assistant, we are going to use a script by tteck which will
automate the process significantly

Proxmox Home Assistant Install ScriptProxmox Home Assistant Install Script

17. Running this script will:
– Find, download and extract the official KVM (qcow2) Home Assistant OS image
– Define user settings, import and attach disk, set boot order and start the VM
automatically
– Install the VM with Default Settings: 4GB RAM, 32GB Storage and 2vCPU cores
– Settings can be tweaked during installation

18. Click your VM on the left and select Shell
19. Copy the following command to run the script and hit enter:

bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/vm/haos-vm.sh)"

Proxmox Home Assistant Install Script Run Proxmox Home Assistant Install Script
Run

20. The wizard will ask you to confirm that you want to create a Home Assistant
OS VM
21. Select Yes and click confirm

Proxmox Home Assistant Install Script ConfirmationProxmox Home Assistant
Install Script Confirmation

22. On the next screen, choose either Default settings or Advanced
– Default settings are fine for a Home Assistant OS install, but you can assign
more RAM and Storage if you need to

23. If you choose Advanced Settings, you will be prompted to choose:
– Home Assistant OS Version (Stable, Beta)
– Hostname (cannot contain underscore ‘_’)
– VM Machine ID
– Machine type
– Allocated CPU Cores
– Allocated RAM Memory

Proxmox Home Assistant Install Advanced Settings CPU CoresProxmox Home
Assistant Install Advanced Settings CPU Cores
Proxmox Home Assistant Install Advanced Settings RAM MemoryProxmox Home
Assistant Install Advanced Settings RAM Memory

24. Select the final Yes to confirm
25. Wait for the script to download, extract and install the latest KVM image
of HA OS
26. Once you get a Completed Successfully message, HA is installed

Proxmox Home Assistant Install Script FinishedProxmox Home Assistant Install
Script Finished

27. To see the IP address your router assigned to your Home Assistant VM
instance, click your node on the left
28. Select your newly created Home Assistant VM
29. The IP address is displayed in the middle
– Use it to access Home Assistant in your web browser 192.168.xxx.xxx:8123

Proxmox Home Assistant Install Welcome ScreenProxmox Home Assistant Install
Welcome Screen

30. Finished, you just installed HA OS on Proxmox!

(Optional) USB Passthrough to Home Assistant VM

If you use a Zigbee or a Z-Wave dongle, there is one additional step you must
do in order to allow the Home Assistant VM to read your attached USB device.

1. Attach your USB Device (Zigbee dongle) to your Mini PC
2. Select your Home Assistant VM on the left
3. Select Hardware from the Menu
4. Click Add at the top bar
5. Select USB Device

Proxmox Home Assistant USB Passthrough Zigbee Z-Wave DongleProxmox Home
Assistant USB Passthrough Zigbee Z-Wave Dongle

6. Select Use USB Vendor/Device ID from the menu
7. Select your USB Device (Zigbee/Z-Wave Dongle)
8. Click Add

Proxmox Home Assistant USB Passthrough Zigbee Z-Wave DongleProxmox Home
Assistant USB Passthrough Zigbee Z-Wave Dongle

8. At the top corner, press the small arrow next to Shutdown
9. Choose Reboot and confirm to restart the VM
10. If you navigate to Home Assistant, you USB dongle will be auto-discovered

If you are having issues, read how to properly Passthrough USB Devices To Home
Assistant.

(Optional) Post Install Proxmox VE 7 Script

This step is optional, but I highly suggest you run this script too as it will
clean up your Proxmox Virtual Environment, get rid of the Enterprise message
that keeps pestering you, add or correct repositories and update Proxmox VE.

Proxmox Home Assistant Post Install Clean up and optimizationProxmox Home
Assistant Post Install Clean up and optimization

To run this script, select your node on the left and click Shell again:

1. Copy the following code in the Shell and hit enter

bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"

2. You will get a couple of confirmation prompts
3. Answer “y” to all of them and finally confirm to reboot Proxmox

Proxmox Home Assistant Post Install Clean up and optimization confirm all
Proxmox Home Assistant Post Install Clean up and optimization confirm all

4. Done

Summary

Deploying Home Assistant on a Proxmox Virtual Environment using these scripts
is a breeze. Developer tteck made it a seamless process that even the
non-technical can follow. He has a bunch of scripts on his repository, which
allow you to further customize your Home Assistant installation in a VE. For
example:

  • Install Zigbee2MQTT as a separate LXC
  • Install MQTT Mosquitto Broker as separate LXC
  • Install ESPHome as a separate LXC
  • Install MariaDB as a separate LXC
  • Install InfluxDB as a separate LXC
  • Deploy AdGuard or Pi-Hole for system-wide ad blocking
  • Deploy Tailscale for easy remote access

The advantage of deploying these Home Assistant add-ons separately is they will
be independent of Home Assistant and it’s current state (installing, rebooting,
breaking changes etc). You can connect them together and setup HA to read data
from each one, operating as it should. I intend to cover the process in a
separate tutorial.

If you are looking to replace that old Raspberry Pi, this is the way to go,
especially if you are not familiar with virtual machines and Linux
environments. Consider buying him a coffee for creating these awesome scripts.

UPDATE 29.03.2023: A pro tip for setting the disk cache option on HAOS to
writethrough greatly lowers the chance of a corrupted database on Proxmox.
Thanks Derek!

UPDATE 10.08.2023: You can now easily upgrade to Proxmox 8.0 without loosing
data, keeping all your VMs and containers intact. Follow the upgrade guide.

Categories Guides Tags Devices, Home Assistant, Proxmox

74 thoughts on “How To Install Home Assistant on Proxmox: The Easy Way”

 1. [svg][7526ba]
    Jose
    March 22, 2023 at 5:18 am

    Amazing and useful post

      □ [svg][229807]
        SHS
        March 22, 2023 at 7:46 am

        Thank you!

 2. [svg][8da479]
    tteckster
    March 23, 2023 at 7:39 pm

    Going forward, versioning will no longer be utilized in order to avoid
    breaking web-links in blogs and YouTube videos (for the final time). So, as
    a result, the name of the shell script has been updated from `haos-vm-v5.sh
    ` to `haos-vm.sh` .

      □ [svg][229807]
        SHS
        March 23, 2023 at 7:43 pm

        Hey,

        Thank you for stopping by.
        I will update the code and thanks again for creating these awesome
        scripts!

        Cheers!

 3. [svg][e4e3c6]
    Derek
    March 28, 2023 at 9:25 pm

    Databases can be very sensitive to sudden power loss. This is less of an
    issue on bare metal, but is a big concern in virtual environments. The
    hypervisor may do caching/buffering that the DB is not expecting and can
    result in irrecoverable DB corruption. I would strongly urge Proxmox users
    to set the disk cache option on the HAOS VM to “writethrough” to greatly
    lessen the chance of lost writes that cause corruption.

      □ [svg][229807]
        SHS
        March 29, 2023 at 7:41 am

        Thank you for the tips Derek, I will add them to the post.
        Cheers!

      □ [svg][dec0ef]
        Peter
        October 24, 2023 at 11:51 pm

        where do i find that option ?

 4. [svg][e4e3c6]
    Derek
    March 28, 2023 at 9:27 pm

    I would also suggest adding an optional step to opt-in to the latest 6.x
    kernel that Promox offers. This can really help with hardware compatibility
    /drivers, particularly on newer Intel platforms.

 5. [svg][2fa441]
    Jezza
    April 4, 2023 at 5:58 pm

    Me getting to the end and there is no zigbee2mqtt instructions :.(

      □ [svg][229807]
        SHS
        April 4, 2023 at 6:09 pm

        Hey, I decided to write a dedicated tutorial for separating Zigbee2MQTT
        from Home Assistant in Proxmox.
        It involves covering and fixing a lot of potential issues.
        Stay tuned, will release it in a couple of days.

        Cheers!

      □ [svg][229807]
        SHS
        April 6, 2023 at 12:37 pm

        Hey Jezza,

        Here it is:

        https://smarthomescene.com/guides/
        how-to-separate-zigbee2mqtt-from-home-assistant-in-proxmox/

        Cheers!

 6. [svg][d734df]
    Kevin
    April 7, 2023 at 12:49 pm

    The script produces an error on Proxmox 7.4-3:
    Using Default Settings
    Using HAOS Version: 9.5
    Using Virtual Machine ID:
    Using Machine Type: i440fx
    Using Disk Cache: Default
    Using Hostname: haos9.5
    Using CPU Model: Default
    Allocated Cores: 2
    Allocated RAM: 4096
    Using Bridge: vmbr0
    Using MAC Address: 02:63:F2:9B:5E:DE
    Using VLAN: Default
    Using Interface MTU Size: Default
    Start VM when completed: yes
    Creating a HAOS VM using the above default settings
    – Validating Storage…bash: line 362: pvesm: command not found
    numfmt: invalid number: ‘\033[01;31m362\033[m:’

    [ERROR] in line 355: exit code 2: while executing command FREE=$(echo $line
    | numfmt –field 4-6 –from-unit=K –to=iec –format %.2f | awk ‘{printf(
    “%9sB”, $6)}’)

      □ [svg][d734df]
        Kevin
        April 7, 2023 at 1:11 pm

        My error – the script ran fine once I used root on Proxmox

 7. [svg][df471a]
    Steve
    April 13, 2023 at 5:01 pm

    Thanks for this guide and thanks to the Dev for the scripts.
    Can I ask I’m wanting to do this mainly for
    HassOS,
    ZigBee2Mqtt & Node red.
    But there’s about another 4 on that list I would like also and each ones
    shows default ram and CPU cores,
    so say I want 8 small VMs that say they need 1gb ram each and 1-2 CPU cores
    would that mean I need a system with 8gb Ram and 8-16 CPU cores 😲
    Or are you not meant to add them together like that.

      □ [svg][229807]
        SHS
        April 14, 2023 at 10:51 am

        No, no need. All containers share the kernel and resources of the PC,
        so you can get away with less.

        It largely depends on what you intend to run though

 8. [svg][7a7dec]
    Shawn
    April 13, 2023 at 10:47 pm

    The final script did not work. I put it in (3x – copied twice and then
    typed a third) – system just went to the next line. No evidence anything
    happened. Ideas??

      □ [svg][229807]
        SHS
        April 14, 2023 at 10:47 am

        Which script are you reffering to?

          ☆ [svg][2b3ab2]
            Liam
            May 1, 2023 at 1:29 am

            I believe he is referring to the wget command bash -c “$(wget -qLO
            – https://github.com/tteck/Proxmox/raw/main/vm/haos-vm.sh)”.

            I am also having the same issue. In the shell entering the above
            command does nothing.

              ○ [svg][229807]
                SHS
                May 2, 2023 at 7:43 am

                Hello,

                The command is correct, I’ve just tested it. Are you sure you
                are pasting it correctly?
                Try typing it out manually and let me know.

 9. [svg][69529c]
    GavC
    April 17, 2023 at 12:00 pm

    Awesome stuff. Thank you.
    What a great tutorial and script. Saved me sooo much time on my rebuild
    compared to my first implementation.
    The only thing I’m having an issue with is the USB device config.
    I’ve config’d the proxmox vm as you did for USB vendor/Device ID and
    rebooted.
    HAS does auto discover the Conbee II and I can see it in integrations but
    when I try to configure it I get an error “Failed to probe the usb device”
    then it disappears from the integrations page until I reboot and it’s auto
    discovered again. I tried a few different ways, incl usb as a passthrough
    port but same error.

    Any advice ?
    Thanks
    Gav
    HAS 2023.4.4
    Proxmox 7.3-6

      □ [svg][229807]
        SHS
        April 18, 2023 at 7:42 am

        Hello,

        Did the Conbee worked before? Seems like a firmware issues, have you
        tried flashing it with the latest firmware?
        I had similar problems with a Sonoff ZBDongle-E which I fixed by
        flashing the latest firm.

        Let me know.

          ☆ [svg][69529c]
            GavC
            April 21, 2023 at 3:19 am

            It did work previously but it was a few yrs ago when I set it up.
            Other research found there were problems with the conbee after a
            firmware update.
            I did flash a few different firmware versions but no joy so I’ve
            now bought a SkyConnect and it’s chugging along nicely.

10. [svg][9fcc8c]
    dngy
    May 2, 2023 at 11:39 am

    I have the same issue, the command does nothing. I tried to copy/paste and
    tried to type it… Any idea why it is not executing? I’m running it as a
    root user.

      □ [svg][229807]
        SHS
        May 3, 2023 at 7:32 am

        Are you sure you are pasting it correctly? If a letter or two is
        missing, it won’t do anything.
        Use Shift + Insert to paste the script, I am saying it again: It is
        correct.

          ☆ [svg][c54f29]
            Mark
            October 29, 2023 at 2:22 am

            I had the same issue. I followed the http link to the script and
            did a copy of the code. Then I opened the console command and
            navigated to the location of where the script was supposed to
            reside and nano created the .sh script with a paste. I then had to
            make it executable and manually ran the script from the cli. It
            worked and was a bit of a fudge. I’m not sure why me and others
            can’t run the script from the wget code.

11. [svg][5647c5]
    kojam
    May 3, 2023 at 3:16 am

    “UPDATE 29.03.2023: A pro tip for setting the disk cache option on HAOS to
    writethrough greatly lowers the chance of a corrupted database on Proxmox.
    Thanks Derek!”

    How? LOL

12. [svg][5647c5]
    kojam
    May 3, 2023 at 3:27 am

    Got it!
    HAOS
    – Hardware
    – Hard Disk (double click to open option window)
    – Cache dropdown
    – Write through

13. [svg][fa2701]
    Rich
    May 3, 2023 at 5:34 pm

    Hello. First I want to say I appreciate the work you have done. I have just
    completed proxmox with the intent of learning virtualization and especially
    home assistant, and many other things. I started researching installing HA
    and found your guide which seems to make it easy.

    However, I like others I have experienced the issue where using the command
    does nothing. There were no available updates on my fresh install (not sure
    if that’s weird or convenient)

    At first I was like ‘well, lets make sure I can reach the outside world’ so
    I used traceroute (after getting irate that shell wouldn’t recognize
    tracert until I remembered I was using linux) and that was successful. I
    rebeooted the machine (step 1, just in case, right).

    I copied and pasted the link several times, copied it to notepad to make
    sure there are no extra spaces at the beginning or end, and typed it
    carefully several times and am still experiencing the issue others have had
    where the command does nothing.

    I’m sure your script does work in 99 percent of the cases and don’t expect
    you to cater to me (especially because you are doing this out of the
    kindness of your heart and to help people) but it does seem to be more than
    an isolated issue. Do you have any suggestions to any configuration issues
    that may be causing a few of us to experience difficulties?

    Either way, thank you for your contributions, I am excited to start the
    fun!

      □ [svg][229807]
        SHS
        May 3, 2023 at 5:38 pm

        Okay, this is such a well-written comment that I would to thank you for
        providing your feedback.

        The shell command has been working without issues for me, BUT there
        might really be something going on since so many people reported it. I
        will dig deeper and post a video tommorow of EXACTLY the steps I take
        to install the VM using the script.

        Cheers

          ☆ [svg][3e835d]
            Rich
            May 3, 2023 at 6:35 pm

            Thank you! In reading through your instructions to set up HA I
            skipped the ProxMox install as I already had it installed. I went
            back and read and the only difference in my install is I used a
            fake FQDN as that’s what the install suggested. I don’t know if
            that will affect this situation, but wanted to give you any info I
            have.

          ☆ [svg][1e8869]
            Scooter
            September 15, 2023 at 5:01 am

            If I understand the problem, I ran in to the same. When I copied
            the script and pasted it into the Proxmox Shell, then hit “Enter”,
            nothing happened. Tried several times with same result; it just
            returns immediately to the command line. I then typed in the script
            manually and hit Enter and it ran just fine. I later ran into the
            same problem with copying and pasting scripts into the Shell where
            nothing happened. But, if I manually type in the script it runs
            fine. Not sure what the problem is, but it is repeatable and I have
            not yet resolved the problem with pasted-in scripts.

14. [svg][5647c5]
    kojam
    May 4, 2023 at 4:09 am

    Your traceroute worked? I’m not sure if that’s what you said. (Forget how
    to traceroute in Linux.)
    It wasn’t working for me either. I ran it several times and thought it was
    to work silently. LOL.
    After a few hours I came back to check and no change. Then I decided to
    ping something. I found out that internet wasn’t working. Fixed that and
    ran the script again.
    IT WAS A THING OF BEAUTY!

    Thanks!!!
    You have NO IDEA how much pain you saved me!

      □ [svg][fa2701]
        Rich
        May 4, 2023 at 6:37 am

        My Traceroute did work (On linux its traceroute, on Windows its
        Tracert). My trace route did work and I still am having the same issue,
        but I can’t help but wonder if I’m having some sort of connection issue
        to the outside because I do get an error trying to find the Debian
        repositories, but it’s weird that I can ping, trace route and still get
        nothing.

15. [svg][229807]
    SHS
    May 4, 2023 at 7:47 am

    To anyone saying the script does nothing: It’s almost certainly a
    connection issue, double check your internet connection.

    https://imgur.com/a/oB3HceJ

      □ [svg][73b9c2]
        RugbyAl
        July 28, 2023 at 9:54 pm

        I mucked up DNS and got these symptoms, i.e. script wouldn’t work, but
        it wouldn’t without DNS would it! All worked well for me – thanks.

16. [svg][fa2701]
    Rich
    May 5, 2023 at 3:28 am

    I’ve racked my brain all day at work, and then I had an epiphany and I
    solved my issue (and I think everyone elses issue) with the script not
    working. I still had the 127.0.0.1 which was the default example for the
    DNS- I only ever pinged and traced the DNS, but never tried a domain. I
    switched my DNS to the google 8.8.8.8 and sure enough, the script worked.

    Not bad for a car salesman who just dabbles in computers for fun, right?

17. [svg][26adec]
    virtM
    May 23, 2023 at 4:45 am

    Awesome, just upgraded from Virtualbox running on a windows 10 PC with a
    script to auto start that was having issues after an internet outage
    requiring the host to be rebooted to regain access. TY TY!

18. [svg][498177]
    Zac
    June 16, 2023 at 7:18 pm

    So helpful, thank you for taking the time to do this.

      □ [svg][229807]
        SHS
        June 16, 2023 at 7:20 pm

        You are welcome, cheers!

19. [svg][cfef52]
    Joost
    June 16, 2023 at 10:37 pm

    great guide!

    maybe you can add on step 23 Hostname cannot consist _

    script will fail, got the info from the home assistant forum

      □ [svg][229807]
        SHS
        June 17, 2023 at 7:29 am

        Thank you, noted.

20. [svg][9841e7]
    Karma
    June 24, 2023 at 6:27 pm

    Hello.
    I added another hard drive in the HA vm.
    But HA can’t use it… How can I do it?
    Thanks
    “

      □ [svg][229807]
        SHS
        June 26, 2023 at 7:35 am

        You can attach it as a network drive using Samba NAS addon

21. [svg][08ca7c]
    GL-user
    June 30, 2023 at 3:58 pm

    Question,
    Is there a reason why the HA is build as a VM and not as a container?
    Just wondering, since I had the option to go through Docker on an Ubuntu
    OS, but I read Proxmox can also handle both.
    I’m using now Proxmox on an HP elite desk 8GRam and i5 7Gen.
    Thanks for a reply.

      □ [svg][229807]
        SHS
        July 1, 2023 at 7:37 am

        Well, HAOS in this case deployed as a VM benefits greatly from the
        supervisor. It can run addons natively, without setting up each one
        separately.

        But if you want, you can deploy it as an LXC too, or just run HA Core
        as an LXC.
        See all the different methods of installation before you decide.
        https://www.home-assistant.io/installation/

22. [svg][727933]
    Aimak
    July 4, 2023 at 9:38 pm

    Count me as a successfully deployed Proxmox+HA+Zigbee in a Lenovo M93p

    So easy and straight forward. Thank you so much!

      □ [svg][229807]
        SHS
        July 5, 2023 at 7:40 am

        You are welcome! Cheers

23. [svg][efcafe]
    GL-user
    July 6, 2023 at 11:56 am

    Ok, thanks. I’ll give it another look before deciding.

24. [svg][350881]
    Roger
    July 8, 2023 at 5:38 pm

    Thank you very much for this. I successfully migrated my HA to a Proxmox VM
    🙂

25. [svg][007b7e]
    Mike
    July 15, 2023 at 1:57 am

    I installed Proxmox using this method on a NUC which was running Windows
    11. Is there any way of reversing the Proxmox installation so that I go
    back to Windows 11 and all the apps that were previously running?

      □ [svg][229807]
        SHS
        July 17, 2023 at 6:43 am

        No, you formatted the drive when you installed Proxmox. You can run
        Win11 in Proxmox too though!

26. [svg][a720b3]
    Luke
    July 29, 2023 at 1:02 pm

    Hi,
    Thank you for your extensive guide. I’ve got it up and running but I’m
    struggling to reach certain port from an addons. In my case I would like
    port 81 to be accessible in order to acces the Nginx Proxy Manager UI, is
    this something that I need to configure on the VM in Proxmox?

    Thank you

      □ [svg][229807]
        SHS
        July 31, 2023 at 8:10 am

        By default, Nginx is accessible from the IP:81
        See here for 443
        https://pve.proxmox.com/wiki/Web_Interface_Via_Nginx_Proxy

27. [svg][594a2e]
    Michal
    August 30, 2023 at 5:10 pm

    Hi,
    if you have problem with download HA from script, just change DNS in
    Proxmox on 8.8.8

28. [svg][915547]
    RMF
    September 18, 2023 at 5:41 pm

    Thank you for this tool. It worked for me the first time, however I decided
    to “blow up” my HA setup and start over – and now having issues.

    I successfully removed/deleted my original VM containing HA.

    Running the script again – getting some of the same issues listed above –
    cutting and pasting the script command the first time results in nothing
    happening. Typing it didn’t help either.

    I simply get a next line with a “>” displayed.

    If I past it AGAIN, then it will run – seems to run successfully. I can see
    it create a new VM which appears under the PVE for a few seconds …. then it
    disappears and I get this error:

    bash: -c: option requires an argument
    [ERROR] in line 431: exit code 2: while executing command bash -c

    Any suggestions here?

      □ [svg][229807]
        SHS
        September 19, 2023 at 9:58 am

        See the other comments bellow, you need to check your DNS settings.
        The second error is most likely because you are not pasting the command
        correctly. Use CTRL + SHIFT + V

      □ [svg][6b5f0a]
        Anthony
        September 22, 2023 at 4:39 pm

        I screwed up my copy paste and got this behavior. I got it to go making
        sure the command had the trailing quote and tossed semicolon on for
        good measure.

29. [svg][6b5f0a]
    Anthony
    September 22, 2023 at 4:47 pm

    Does the cleanup script work with proxmox ve 8?

30. [svg][dbedf3]
    Clay
    September 27, 2023 at 5:10 am

    Hi there, you mention that there’s a script to get rid of the subscription
    error message. Can you provide that?

      □ [svg][229807]
        SHS
        September 27, 2023 at 7:21 am

        Yes, it’s the post install cleanup script:
        https://smarthomescene.com/guides/
        how-to-install-home-assistant-on-proxmox-the-easy-way/#
        optional-post-install-proxmox-ve-7-script

31. [svg][78b81e]
    Randy
    October 16, 2023 at 10:31 am

    Hi, I’m getting an error after install.
    Using google DNS and its working. Did cntr shift v to paste

    Any ideas?

    Using HAOS Version: 11.0
    Using Virtual Machine ID: 101
    Using Machine Type: i440fx
    Using Disk Cache: Write Through
    Using Hostname: haos11.0
    Using CPU Model: Host
    Allocated Cores: 2
    Allocated RAM: 4096
    Using Bridge: vmbr0
    Using MAC Address: 02:AD:74:67:39:03
    Using VLAN: Default
    Using Interface MTU Size: Default
    Start VM when completed: yes
    Creating a HAOS VM using the above default settings
    ✓ Using local-lvm for Storage Location.
    ✓ Virtual Machine ID is 101.
    ✓ https://github.com/home-assistant/operating-system/releases/download/11.0
    /haos_ova-11.0.qcow2.xz
    ✓ Downloaded haos_ova-11.0.qcow2.xz
    ✓ Extracted KVM Disk Image
    ✓ Created HAOS VM (haos11.0)
    ✓ Started Home Assistant OS VM
    ✓ Completed Successfully!

    bash: -c: option requires an argument

    [ERROR] in line 442: exit code 2: while executing command bash -c

      □ [svg][229807]
        SHS
        October 18, 2023 at 7:44 am

        You are either pasting wrong or pasting at the wrong space.
        In any case, update Proxmox packages.
        Reboot the host.
        Paste the command.

        Let me know how it goes

32. [svg][78b81e]
    Randy
    October 20, 2023 at 7:30 am

    Thank you for your reply.
    I ran update, nothing new was found. Rebooted.
    This is what I pasted into the shell:
    bash -c “$(wget -qLO – https://github.com/tteck/Proxmox/raw/main/vm/
    haos-vm.sh)
    I see the VM being created, then deleted after the same error message:

    bash: -c: option requires an argument

    [ERROR] in line 442: exit code 2: while executing command bash -c

33. [svg][1ab58f]
    Chris
    October 25, 2023 at 9:13 pm

    you forget the ” at the end
    you pasted:
    bash -c “$(wget -qLO – https://github.com/tteck/Proxmox/raw/main/vm/
    haos-vm.sh)

    it has to be:
    bash -c “$(wget -qLO – https://github.com/tteck/Proxmox/raw/main/vm/
    haos-vm.sh)”

34. [svg][815118]
    Fred
    October 29, 2023 at 5:38 pm

    Hello from France !

    Many thanks for this very effective tutorial. A very nice job.
    I followed it by directly installing version 8 of Promox on a NUC N95 / 8GB
    DDR5 3200MHz / 256GB nvme SSD for €135. Faster and cheaper than a Raspberry
    Pi 4 (which I love) with power supply + sdd + case

    Congratulations
    Fred

35. [svg][b47685]
    frank smith
    November 5, 2023 at 6:00 pm

    Simplified things. Is it possible to get the scripts. I do like to review
    the scripts to verify security and malicious activity possibilities.

36. [svg][9cd11e]
    Stephen
    November 16, 2023 at 2:52 am

    Hi,
    I succesfully installed PVE and HA as you suggested. I read your article on
    Zigbee, MQTT. If I understand it correctly PVE has to be in a container,
    not a VM. Can the VM be converted to a LXC container? If not how do you
    create the HA container? Thanks for your help.
    Steve

      □ [svg][229807]
        SHS
        November 16, 2023 at 7:26 am

        Hello Stephen,
        PVE stands for Proxmox Virtual Environment and it’s just that – an
        Debian-based virtual environment that you use a base and install
        applications (containers) or virtual machines (operating system) on to
        it.
        You cannot convert a VM to an LXC, you would have to reinstall it.
        Please tell me, ultimately, what are you trying to achieve?

37. [svg][6fad72]
    Daniel
    November 24, 2023 at 10:17 pm

    First time proxmox and HA user so excuse my ignorance. I’ve followed the
    guide no problem and have HA running. I read there is different types of HA
    installs, (full install, hass.io, supervised etc) which one is this or what
    do these terms mean? I was searching for the Myenergi integration but it
    doesn’t show up and supposedly it may not be available depending on the
    type of install. If i want to manually add an integration how can I get to
    the folder directorys in my HA instance on proxmox? It’s probably staring
    me in the face but I can’t find it anywhere. All help greatly appreciated.

      □ [svg][229807]
        SHS
        November 25, 2023 at 7:46 am

        If you followed the article, you installed HAOS (Home Assistant
        Operating System) which is the full version capable of using containers
        as add-ons.
        Assuming you have installed HACS (Home Assistant Community Store), you
        can search Myenergi within HACS and install it.

        See more here: https://github.com/CJNE/ha-myenergi

38. [svg][6d54a4]
    Niclas
    December 28, 2023 at 1:23 pm

    i have Lidl zigbee hub and Skyconnect, and i want to use both at the same
    time. how can i emigrate from lidl to ha?

      □ [svg][229807]
        SHS
        December 29, 2023 at 8:15 am

        You can simply ditch the hub and re-pair everything to SkyConnect in
        Home Assistant.
        You cannot use both at the same time

39. [svg][1e1f37]
    Maic
    January 20, 2024 at 11:17 pm

    Hi!
    I’m trying to install HA with bash -c “$(wget -qLO https://github.com/tteck
    /Proxmox/raw/main/vm/haos-vm.sh)” command but nothing happen. Circle in
    Status ir rolling…

      □ [svg][229807]
        SHS
        January 22, 2024 at 7:35 am

        Are you pasting it correctly in the Shell?
        Use right click or Ctrl+Shift+V
        bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/vm/
        haos-vm.sh)"

Comments are closed.

Smart pick of the week

Aqara Spring Sale 2024Aqara Spring Sale 2024

AQARA SPRING SALE -30%
Amazon UK • Amazon DE
Amazon FR • Amazon IT • Amazon ES

SwitchBot Spring Sale 2024SwitchBot Spring Sale 2024

SWITCHBOT SPRING SALE -30%
US Webstore • EU Webstore
Amazon US • Amazon UK • Amazon DE

SMART DEVICES REVIEWS

  • Tuya Zigbee Round Thermostat model HT-T010 Featured ImageTuya Zigbee Round
    Thermostat model HT-T010 Featured Image
    New Zigbee Thermostat with Sleek Round Design HC-T010March 19, 2024
  • Moes Star Ring Scene Switch Review FeaturedMoes Star Ring Scene Switch
    Review Featured
    Moes Zigbee Star Ring Scene Switch ReviewMarch 13, 2024
  • Tuya Zigbee Fingerbot Plus Button Pusher - Featured ImageTuya Zigbee
    Fingerbot Plus Button Pusher - Featured Image
    Tuya Zigbee Fingerbot Button Pusher ReviewMarch 4, 2024
  • Moes Smart Plug and Energy Meter _TZ3000_yujkchbz ReviewMoes Smart Plug and
    Energy Meter _TZ3000_yujkchbz Review
    Moes Zigbee Smart Plug with Energy Meter ReviewFebruary 26, 2024
  • AirGradient ONE Review and ESPHome Integration in Home Assistant: Featured
    ImageAirGradient ONE Review and ESPHome Integration in Home Assistant:
    Featured Image
    AirGradient ONE Review and ESPHome IntegrationFebruary 21, 2024
  • Tuya Zigbee 12-Channel Relay Switch Board Featured Image SmartHomeSceneTuya
    Zigbee 12-Channel Relay Switch Board Featured Image SmartHomeScene
    Tuya Zigbee 12 Channel Relay Board ReviewFebruary 16, 2024
  • Nanoleaf Thread Matter Downlight NL65E100 Review: Featured Image
    SmartHomeSceneNanoleaf Thread Matter Downlight NL65E100 Review: Featured
    Image SmartHomeScene
    Nanoleaf Matter-over-Thread Downlight NL65E100 ReviewFebruary 3, 2024
  • Sonoff Mini Extreme MINIR4M Matter Smart Switch Featured Image
    SmartHomeScene.comSonoff Mini Extreme MINIR4M Matter Smart Switch Featured
    Image SmartHomeScene.com
    Sonoff Matter MINIR4M Smart Switch ReviewJanuary 25, 2024
  • Apollo AIR-1 Air Quality Monitor Review: Featured image on
    SmartHomeScene.comApollo AIR-1 Air Quality Monitor Review: Featured image
    on SmartHomeScene.com
    Apollo AIR-1 Review: Complete Air Quality Monitoring for Home Assistant
    January 11, 2024
  • Thirdreality Matter Night Light Featured ImageThirdreality Matter Night
    Light Featured Image
    Thirdreality Matter Night Light ReviewJanuary 5, 2024

SmartHomeScene Logo
     
Ko-Fi SmartHomeScene.comKo-Fi SmartHomeScene.com

QUICK LINKS
Blog
News
Guides
Reviews
DIY

HOME ASSISTANT
Add-ons
Integrations
Cards
Dashboards
Proxmox

DEVICE REVIEWS
Zigbee
Wi-Fi
Bluetooth
Thread
Matter

SITE LINKS
About
Contact
Support
Privacy Policy
Review Guidelines

SmartHomeScene © 2024
All Rights Reserved
 
Close

  • Blog
  • News
  • Guides
  • Home Assistant
      □ Add-ons
      □ Integrations
      □ Cards
      □ Dashboards
      □ Proxmox
  • Device Reviews
      □ ZigBee
      □ Wi-Fi
      □ Bluetooth
      □ Thread
      □ Matter
  • DIY

Search for:
[                    ]
