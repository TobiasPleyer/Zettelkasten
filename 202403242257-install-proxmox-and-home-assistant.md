---
date:  Sunday, March 24, 2024
tags:
---

# Install Proxmox and Home Assistant

[Original](https://www.derekseaman.com/2023/10/home-assistant-proxmox-ve-8-0-quick-start-guide-2.html)

Skip to content
Derek Seaman's Tech Blog

  • Home
  • Smart Home
  • Security
  • About
  • Contact

Derek Seaman's Tech Blog
Main Menu

  • Home
  • Smart Home
  • About
  • Contact

Home Assistant: Proxmox VE 8.1 Quick Start Guide

October 21, 2023
in Smart Home
Tags: Home AssistantProxmoxProxmox 8

Hot off the press is Proxmox VE 8.1, which GA’d in late November, 2023. It is
based on Debian Bookworm 12.2, and has a number of new features like defaulting
to Linux Kernel 6.5. This post is a completely refreshed version of my popular
Home Assistant: Proxmox VE 8.0 Quick Start Guide, but all new for Proxmox VE
8.1. 

Just like my 7.4/8.0 guide, this posts covers installing Home Assistant OS
(HAOS). HAOS is, IMHO, the preferred method to run Home Assistant for the large
majority of people. There are other installation flavors, such as supervised or
container, but those are out of scope for this post. HAOS makes Home Assistant
as turn-key as possible.

Proxmox VE 8.1 has a new text based fall-back installation UI, which can be
very helpful for working around issues with newer CPUs. I did run into GUI
installation issues with Proxmox VE 7.4 on my Intel 12th Generation CPUs, so
this enhancement is very welcome. We will use the new console install method in
this updated post.

Update January 25, 2024: Minor edits. Added that a wired connection is needed
(no Wi-Fi), and modified the hostname FQDN. 

Update December 2, 2023: Added new sections: Two Factor Setup, Proxmox 8.1
Notifications, Glances Monitoring, Proxmox Monitoring. 

Update November 27, 2027: Made a couple of minor tweaks to account for Proxmox
8.1 changes/enhancements. 

Update October 21, 2023: I refreshed this post for the latest version of Home
Assistant and tteck’s awesome scripts. 

If you are already running Proxmox VE 7.4 and want to upgrade, check out my
guide:

How-to: Proxmox VE 7.4 to 8.0 Upgrade

[2023-04-02_09-25-29-1024x170]

More Home Assistant Related Posts

I’ve written a number of posts related to Home Assistant which you might find
useful:

Home Assistant: Getting Started Guide
Home Assistant: Ultimate Backup Guide
Home Assistant: Ultimate Restore Guide
Home Assistant: Monitor Proxmox with Glances 
Home Assistant: Installing InfluxDB (LXC)
InfluxDB 1.x Automated Backups
Home Assistant: InfluxDB Data Management (LXC)
Home Assistant: Installing Grafana (LXC) with Let’s Encrypt SSL
InfluxDB + Chronograf: Configuring Let’s Encrypt SSL

What's covered in this guide?

This guide walks you through a bare metal installation of Proxmox, followed by
deploying a Home Assistant OS (HAOS) VM. To be more specific I cover:

  • Why Proxmox VE for Home Assistant?
  • Proxmox Storage Recommendations
  • Creating Proxmox USB Boot Media
  • Installing Proxmox VE 8.1
  • Proxmox Post-Install Configuration
  • Intel Microcode Update (Optional)
  • Installing Home Assistant OS (HAOS) VM
  • Setting Static IP Address (Recommended)
  • Blocked DNS over HTTPS Workaround
  • USB Passthrough to HAOS (Optional)
  • Optimize CPU Power (Optional)
  • Check SMART Monitoring (Optional)
  • VLAN Enable Proxmox (Optional)
  • Proxmox Let’s Encrypt SSL Cert (Optional)
  • Proxmox Two Factor Setup (Optional)
  • Proxmox 8.1 Notifications (Optional)
  • Glances Configuration (Optional)

Why Proxmox VE for Home Assistant?

First, why would you run Home Assistant OS as a VM on Proxmox VE 8? Well, Home
Assistant is typically not very resource hungry and even old mini-PCs from
several years ago would have a lot of left over computing resources that you
can run other services. Running Proxmox VE, which is a free hypervisor based on
KVM, has a nice management UI and is pretty easy to use. It allows you to run
HAOS in a VM, and you can then run other VMs or LXC containers on the same
hardware. 

If you are new to Home Assistant, not super nerdy, and just want a super
reliable and easy to use “appliance”, then don’t go the Proxmox VE route. Just
grab a cheap used mini/ultra-mini PC and run Home Assistant OS on it “bare
metal” and be done with it. But if you know you want HAOS as a VM, potentially
do LXC containers down the road, Proxmox VE is a great (and free) option. Even
though Home Assistant can do backups, being able to do a whole HAOS VM snapshot
at the hypervisor level can be great for roll-back from failed upgrades or “oh
crap” mistakes.

The process covered in this post uses the awesome tteck Proxmox scripts. This
makes the process super easy, grabs the latest version of HAOS, and enables
GUI-driven advanced customizations. He’s updated his scripts to be compatible
with Proxmox VE 8.1.

[2023-06-23_11-23-38-1024x574] [svg] Image Courtesy Proxmox.com

Proxmox Storage Recommendations

Proxmox is designed to be used on everything from an old PC to high-end
enterprise production environments. There is certainly some debate on what type
of storage configuration you should use. For example, should I use ZFS? Should
I use multiple physical storage devices and use ZFS mirroring? Do I need a
separate boot drive? Separate log and cache drives? 

Those are all valid questions to ask in an enterprise production environment.
However, for a home environment my thinking is KISS (keep it simple stupid),
unless you REALLY know what you are doing. Why overly complicate your
configuration for a home server? 

For a home environment where you have a NAS (such as Synology, QNAP, etc.) I
would suggest: 

  • Use a single SSD/NVMe (likely M.2) drive in your Proxmox server. Make sure
    it’s large enough for future growth. 
  • Use the EXT4 (default) filesystem with LVM-thin (also default)
  • Use Proxmox’s built-in backup to do nightly backups of all VMs and LXC
    containers to your NAS
  • Use Home Assistant’s Google Drive backup add-on to do nightly backups to
    the cloud
  • Configure Home Assistant 2023.6 (and later) to backup directly to your NAS

For more information about Home Assistant Backups and Restores, check out my
posts:

Home Assistant: Ultimate Backup Guide
Home Assistant: Ultimate Restore Guide

Super nerds may want to deviate from using a single drive in their Proxmox
server and throw in ZFS, do ZFS mirroring, and use multiple storage devices.
More power to them if that’s what they want. But for the vast majority of Home
Assistant users, a single drive with robust backup is more than sufficient.

Do NOT try and mount NAS storage to the Proxmox host and use that for VM or LXC
storage. Always build your VMs or LXCs on local Proxmox storage, or risk
running into significant issues.

Creating Proxmox USB Boot Media

In order to install Proxmox on your server, we need to create a bootable USB
drive. This is super easy and you can use your favorite tool like Balena Etcher
(Mac/PC/Linux). The Proxmox VE installer ISO is very small, at just a bit over
1GB, so your boot media need not be large. 

 1. Download the latest “Proxmox VE 8.1 ISO Installer” 
 2. Download and install Balena Etcher
 3. Insert your USB boot media
 4. Launch Balena Etcher and select Flash from file 

[2023-04-02_09-43-59-1024x614] [svg]

5. Locate the Proxmox VE ISO you downloaded
6. Click on Select target and select your USB boot media

[2023-04-02_09-48-22-1024x614] [svg]

7. Click on Flash to write the image to the boot media. On a Mac you may be
prompted for your password. 
8. Wait a minute or so for the image to be written and verified.



[2023-06-23_08-50-11-1024x614] [svg]
[2023-06-23_08-51-19-1024x614] [svg]

9. After the image is written close Balena Etcher and remove your USB  boot
media. On a Mac it is automatically safely dismounted so just pull it out.

Installing Proxmox VE 8.1

 1. Connect a keyboard and monitor to your Proxmox server/mini-pc/NUC/etc. Plug
    in an ethernet cable. Do not use Wi-Fi. 
 2. Power off your server and insert the USB boot media
 3. Power on your server and press the right key to enter your BIOS setup
    (varies by manufacturer)
 4. Depending on what OS you were previously running, a few settings might need
    to be tweaked. The name of these settings and menu location varies by BIOS
    manufacturer. Review the following settings:

  • Enable Virtualization (may be called VT-x, AMD-V, SVM, etc.)
  • Enable Intel VT-d or AMD IOMMU (Future proofing for PCIe/GPU passthrough)
  • Leave UEFI boot enabled. Enable secure boot if installing Proxmox 8.1 or
    later. Disable secure boot if installing Proxmox 8.0 or earlier.
  • Enable auto-power on (Ensures host will power on after a power
    interruption)
      □ This setting can be hard to find, have non-obvious names (e.g. setting 
        State after G3 to S0 State), or not exist at all. Varies by
        manufacturer. See screenshot below.
  • If you think down the road you might add a PCIe card (like a m.2 Google
    Coral TPU), make sure you look through the BIOS settings and DISABLE any
    PCIe power management (ASPM) options you find.  
  • Change the boot order to set your USB boot media at the top.

[IMG_0119-1024x717] [svg] Enable Auto Power On
[IMG_0120-1024x715] [svg] Disable PCIe Power Management (Google Coral)
[IMG_0121-1024x715] [svg] Enable Secure Boot (Proxmox 8.1 and later)

5. Save the BIOS settings and reboot. If all goes well, Proxmox VE 8.1
installer will start up.

[2023-11-27_20-23-02-1024x809] [svg]

Note: New to Proxmox VE 8.0 and later is a Terminal UI installation option.
This will gather all the required information for the install, except without
using the graphics card. Why can this be helpful? Sometimes the Proxmox
installer has compatibility issues with the graphics card. I would suggest
trying the Graphical method, and if that fails, switch back to Terminal UI.
With Proxmox 8.1 on my 12th Gen Intel the graphical method now works, unlike
Proxmox 8.0.0 I’ll walk you through the Terminal UI method, but the graphical
method asks the same questions.

6. Arrow down to Install Proxmox VE (Terminal UI).
7. Press enter on I agree on the EULA
8. Select the Target harddisk and press enter on Next.

Note: Don’t change the filesystem unless you know what you are doing and want
to use ZFS, Btrfs or xfs. The default is EXT4 with LVM-thin, which is what we
will be using.

9. Select your Country, Time zone and Keyboard Layout.

[2023-11-27_20-32-46-1024x479] [svg]

10. Enter a strong root password and an email address.

11. Select your Management Interface, Hostname, IP address, Gateway and DNS
Server.

Note 1:  If your server is plugged into the network it should grab a DHCP
address and populate the other information. I strongly recommend either using a
static IP, or create a DHCP reservation for this server. You don’t want the IP
to change on you. 

Note 2: Put some thought into the Proxmox hostname you want to use. YOU CAN’T
change this later or Proxmox will have serious (likely unrecoverable) problems.
I’d use something generic like proxmox1.local.  

[2023-11-27_20-35-14-1024x490] [svg]

12. Triple check everything on the Summary screen is correct then select
Install. 

[2023-11-27_20-36-10-1024x579] [svg]

Proxmox Post Install Configuration

Before we install Home Assistant, we need to do a couple of configuration
tasks. First, we need to update Proxmox with the latest packages. Please note
you MUST run the post-configuration script before you do any Proxmox updates or
deploy HAOS. If not, you will likely see 401 errors with the enterprise
repositories since you (likely) don’t have a Proxmox paid license.

 1. Open a browser and navigate to the IP address and port 8006 (e.g. https://
    10.13.2.200:8006). Click through all your browser warnings and connect
    anyway.
 2. Login with the root username the password you selected during the install
    process.
    Note: You will get a subscription warning. We will fix that in a second.
    Acknowledge the warning.

[2023-04-02_10-56-50] [svg]

3. In the left pane click on the hostname of your Proxmox server.
4. Click on Shell in the middle pane and paste in the following command to run
the awesome tteck post install script:




bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"




[2023-06-23_10-43-17-1024x332] [svg]

5. The tteck script will ask you a series of questions. Run the script and
answer Y to all of the questions. Wait a few minutes for all of the updates to
install.

[2023-06-23_10-44-06-1024x535] [svg]
[2023-11-27_21-31-18] [svg]

6. When asked if you wish to reboot press enter on yes.

Intel Microcode Update (Optional)

Intel releases new microcode for their processors from time to time. This is
different from BIOS firmware, as the Intel microcode runs inside the processor.
It can fix CPU bugs or make other changes, as needed. If you are running on an
Intel system you can use the following tteck script to download the latest
Intel microcode and install it. A reboot will be needed for the microcode to
take effect. 

Note: Sometimes the firmware list can be confusing, listing several options. I
chose the latest firmware package, and I select the latest “deb” variant. Not
all CPUs get the same firmware updates. So even if you chose, for example, the
20231114 version, after you reboot the Proxmox host the firmware your CPU is
using might have an older date. This is OK. 




bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/microcode.sh)"




[2023-12-02_08-44-56] [svg]
text text

After the Proxmox host reboots you can run the following command to see if any
microcode update is active. Not all CPUs need microcode updates, so you may
well not see anything listed.




journalctl -k | grep -E "microcode" | head -n 1




[2023-11-21_09-46-40-1024x75] [svg]

Installing Home Assistant OS (HAOS) VM

For the HAOS installation we will be using the awesome tteck Proxmox Helper
Scripts. 

[2023-04-02_11-28-40-1024x758] [svg] tteck Proxmox Helper Scripts

 1. Login to Proxmox and select your server in the left pane.
 2. Click on Shell in the middle pane.
 3. Enter the following command to start the HAOS install via the tteck Proxmox
    script.
    Note: We will be using the advanced settings to optimize the configuration.
    The script automatically downloads the latest HAOS stable image, creates
    the Proxmox VM, and will configure the hardware and networking. 




bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/vm/haos-vm.sh)"




4. Press Enter to proceed.

[2023-04-02_11-33-59-1024x421] [svg]

5. Select Advanced (NOT Yes).

[2023-04-02_11-35-05-1-1024x407] [svg]

6. For the version chose latest stable version. HAOS is regularly updated, so
the screenshot may not reflect what you see. 

[2023-10-21_14-22-34] [svg]

7. Leave the default Virtual Machine ID.
8. Leave the default Machine Type.
9. Verify Write Through Disk Cache is selected. 

Explanation: If you use “none” then the HAOS filesystem or database(s) within
HAOS can be corrupted during unexpected power loss to your server. Write
through causes an fsync for each write. It’s the more secure cache mode, as you
can’t lose data but it is also slower. In fact, Proxmox documentation gives you
a warning for “None” (which is the Default): Warning: like writeback, you can
lose data in case of a power failure.

graphical user interface, application, Word graphical user interface,
application, Word

10. Set the VM’s Hostname.
11. Verify CPU Model is Host.

Explanation: If you have a standalone Proxmox host (i.e. no multi-node
clustering with Live Migration), use Host mode for the CPU Model. The KVM64
model hides some instructions such as MMX, AVX or AES instructions. This can
have a performance impact on the VM. Per QEMU documentation: This [Host mode]
passes the host CPU model features, model, stepping, exactly to the guest. This
is the recommended CPU to use, provided live migration is not required.

graphical user interface, application, Word graphical user interface,
application, Word

12. Chose your Core Count (2 is fine).
13. Chose your RAM (I’d recommend 4-6GB).
14. Leave the Bridge.
15. Leave the MAC Address.
16. Leave the VLAN.
17. Leave the MTU Size.
18. Select Yes to start VM after creation.
19. Select Yes on Advanced Settings Complete.
20. If prompted to select the storage use the space bar to select the desired
disk then tab to Ok.

Note: I STRONGLY urge you to use local Proxmox storage (e.g., local-lvm), not
storage backed by a NAS. If there’s a network hiccup or the NAS goes offline
you could easily end up with a corrupted VM. 

21. Wait for the installer to complete. This will take a few minutes. 
22. Once the VM is created, click on it in the left pane and then select
console in the middle pane. Note the IP address and port number (8123).

text text
text text

23. Open a browser and open a HTTP connection (http://IP:8123).
24. Depending on the speed of your server, you may see a Preparing Home
Assistant screen for several minutes. Wait until you see Welcome!. 
25. Click on Create my smart home.

graphical user interface, application graphical user interface, application

26. On the Create User screen enter your name, username and password. Click
Create Account.
27. On the Home Location screen enter your home address. The map should now
show your home neighborhood. Click Next.
28. Select your Country. Click Next.

graphical user interface, application graphical user interface, application

29. Select what, if any, analytics you went to send back to the Home Assistant
mother ship. Click Next.

graphical user interface, text, application graphical user interface, text,
application

30. On the next screen HA will show you all of the devices and services that it
found on your network. Click Finish. 

graphical user interface, application graphical user interface, application

Setting Static IP Address (Recommended)

I would strongly recommend that HAOS be configured to use a static IP address.
You can do this by either a DHCP reservation in your router, or set a static IP
in the Home Assistant user interface. 

 1. In the left pane click on Settings -> System -> Network. Click on IPv4. 
 2. Change to Static and enter the details.
 3. Change the address you are connecting to in your bowser and verify HA is
    using the new IP.

Matter uses IPv6, so I would suggest keeping IPv6 set to Automatic and NOT
Disabled. 

Blocked DNS over HTTPS Workaround

If you are blocking OUTBOUND DNS over HTTPS, then you might run into a HA bug
where HA floods your firewall with TENS OF MILLIONS of DNS queries
indefinitely. In order to work around this bug, we will install the SSH add-on
and then modify the HA DNS configuration. 

 1. Click on your name in the lower left panel.
 2. In the right pane enable Advanced Mode.
 3. Click on Settings in the left pane, then Add-ons.
 4. Click on Add-on store in the lower right
 5. Search for “ssh” and select the Advanced SSH & Web Terminal. Click on it.
 6. Click Install.
 7. After it installs, click on Show in sidebar.
 8.  Click on Configuration at the top of the screen. 
 9. Enter the username and password that you configured on the initial HA setup
    screen. Click on Save.

[2023-06-19_09-37-30-1024x366] [svg]

10. Click on the Info tab at the top left of the screen. 
11. Click on Start.
12. In the left pane in HA click on Terminal.
13. Run the following two commands:




ha dns options --fallback=false
ha dns restart




14. Re-enable the DoH block rule on your firewall.
15. Monitor your network to verify your firewall is not being flooded with DNS
requests.

USB Passthrough to HAOS (Optional)

If you have any USB dongles for radios such as z-wave, Zigbee, Thread, etc, we
need to pass it through to the HAOS VM. This is optional, and not needed if you
have no USB devices to passthrough. I am using the Home Assistant SkyConnect
Zigbee/Thread/Matter USB dongle.

 1. Connect your USB dongle to your server.
 2. In the left pane click on your HAOS VM.
 3. In the middle pane click Hardware.
 4. Click on Add then USB Device.
 5. Select Use USB Vendor/Device ID.
 6. Chose the USB device to passthrough.

[2023-04-02_13-34-25] [svg]

7. In the upper right hand corner of the VM pane click on the down arrow next
to Shutdown and select Reboot.
8. Home Assistant will auto-discover the dongle after the reboot.

[2023-04-02_13-37-23] [svg]

Optimize CPU Power (Optional)

Depending on your server hardware and how concerned you are about power
consumption, you might want to tweak how Proxmox handles CPU scaling. I have a
12th Generation Alder Lake CPU (i5-1235U), which idles at 8w-10w according to a
smart power plug. 

Run the tteck script below and it will show you what your current scaling
governor is, and you can elect to change it. Make sure you select the enable
cron job to make it persist across reboots.




bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/scaling-governor.sh)"




[2023-10-21_14-49-03] [svg]
graphical user interface graphical user interface

Check SMART Monitoring (Optional)

Proxmox should enable SMART disk monitoring by default. But it’s a good idea to
check the SMART health stats to make sure your media isn’t having any issues.
On rare occasions a motherboard may not support SMART, so it’s always best to
check that it is working.

Check SMART Health:

 1. In the left pane change to Server View. Click on your Proxmox host.
 2. In the middle pane click on Disks.
 3.  In the right pane you should see your disk(s) and SMART status. You can
    click on the main disk device then click Show SMART values to further
    inspect the health.

[2023-05-02_06-26-53-1024x221] [svg]

VLAN Enable Proxmox (Optional)

Out of the box Proxmox is not VLAN aware. Even if you don’t need VLANs right
now, you can still enable a check box to make Proxmox VLAN aware. Then down the
road when you want to setup VLANs, it’s one less thing to remember to do and
VLANs will just work.

To enable VLANs:

 1. In the left pane change to Server View. Click on your Proxmox host.
 2. In the middle pane click on Network (under system).
 3. Click on Linux Bridge (vmbr0). Click on Edit.
 4. Tick the box next to VLAN aware and click OK.

Tip: If you install other services on your Proxmox server like HomeBridge,
Scrypted, etc. then turning on VLAN enable might cause an issue with IGMP
snooping on your physical switches. I had to disable IGMP snooping on my QNAP
switch or Matter would become unstable. However, a newer TP-Link business class
switch had no such issues so using IGMP/MDL snooping worked without a hitch.
This would only happen with buggy switch firmware, so you shouldn’t run into
the issue.  
[2023-04-06_18-17-43-1024x448] [svg]

Proxmox Let's Encrypt SSL Cert (Optional)

If you want to secure Proxmox with a trusted SSL certificate and even add
Proxmox as an iFrame to Home Assistant follow my post Proxmox Let’s Encrypt
SSL: The Easy Button

Proxmox Two Factor Setup (Optional)

Enhancing the security for your Proxmox accounts should be a priority.
Thankfully Proxmox makes it easy to add Two Factor authentication. 

 1. In the left Pane click on Datacenter.
 2. In the middle pane under Permissions click Two Factor.
 3. Click on Add, TOTP.
 4. Add a description at the top, then scan the QR code with your password
    manager (1Password recommended). At the bottom of the window enter the
    validation number. 
 5. Click Add again. Click Recovery Keys.
 6. Click Add.
 7. Copy the recovery keys to your password manager.

Proxmox 8.1 Notifications (Optional)

New to Proxmox 8.1 is centralized notification configuration. You now have one
place to setup SMTP, Sendmail and Gotify notifications.

 1. In the left pane click on Datacenter.
 2. In the middle pane at the bottom click on Notifications.
 3. In the right pane click on Add.
 4. Chose the notification method you want to configure. I used SMTP.
 5. Configure the parameters as needed, and test. 

Proxmox Monitoring (Optional)

If you want to monitor the Proxmox server status (uptime, CPU usage, package
updates, etc.), then use the HACS Proxmox add-on. This compliments the Glances
Monitoring section below. 

Glances Monitoring (Optional)

If you want to monitor the hardware health and performance of your Proxmox
hosts in Home Assistant, you can use Glances. It’s a free tool that can be
easily setup in just a few minutes. Follow my guide below to setup Glances.
This will surface a lot of hardware sensor information such as temperature for
each of your CPU cores, and many other stats. This compliments the Proxmox
monitoring discussed in the previous section. 

Home Assistant: Monitor Proxmox with Glances

Summary

You now have Home Assistant OS (HAOS) running on Proxmox VE 8.1! The whole
process is pretty easy and with the tteck scripts, makes creating the HAOS VM a
snap. Tteck has a bunch of other Proxmox scripts to setup a variety of VMs and
LXC containers. Visually Proxmox VE 8.1 is pretty much the same as 8.0.
However, the new console setup process bypasses any possible graphical install
issues with newer generation GPUs. This is a welcomed change. Plus now that the
Linux Kernel 6.5 is the default, makes the installation that much quicker and
easier. 

Print Friendly, PDF & EmailPrint Friendly, PDF & Email
Buy me a coffee

Related Posts

 
Part 1: Ruckus One Home Wi-Fi Best Practices Guide
Ruckus

Part 2: Ruckus One Home Wi-Fi Best Practices Guide

March 18, 2024
 
Part 1: Ruckus One Home Wi-Fi Best Practices Guide
Ruckus

Part 1: Ruckus One Home Wi-Fi Best Practices Guide

March 18, 2024
 
Smartwings Thread/Matter Blinds Review
Smart Home

Smartwings Thread/Matter Blinds Review

March 2, 2024
 
Ruckus One: Provisioning ICX Switches
Ruckus

Ruckus One: Provisioning ICX Switches

February 23, 2024
Subscribe
Notify of
[new follow-up comments    ]
[                    ]
[›]
Label [                    ]
{} [+]
[                    ] Name*
[                    ] Email*
[                    ] Website
[*] [Post Comment]
Label [                    ]
{} [+]
[                    ] Name*
[                    ] Email*
[                    ] Website
[*] [Post Comment]
14 Comments
Oldest
Newest Most Voted
Inline Feedbacks
View all comments
Steve
December 12, 2023 8:50 am

This is just an absurdly great tutorial — thanks SOOO much for posting this!!

0
Reply
David
December 26, 2023 2:29 am

Amazing tutorial!!! Saved me a lot of time!

I have a question – the default IP that the proxmox gave was IPv6 and not IPv4.
Could you please give a comment about it?

0
Reply
John
January 5, 2024 12:50 pm

It’s my first time I’ve heard about proxmox and tried to switch from docker to
proxmox just for fun and this tutorial just did everything for me!

0
Reply
Marc Odez
January 6, 2024 8:34 am

Great tutorial! But not a single word about storage partitioning (amount,
purpose, recommended size, enz..)?

0
Reply
Allan Bruhn
January 18, 2024 8:24 am

thx really help me

0
Reply
Kiki Bouba
January 24, 2024 10:12 am

This is incredible! I don’t know how I would have done this without this guide!
A couple of snags I hit:
* The server needs an wired ethernet connection for setup.
* You suggested “proxmox1” as a hostname, which is invalid. “proxmox1.local”
works, though.

0
Reply
Author
Derek Seaman
January 25, 2024 7:47 pm
Reply to  Kiki Bouba

Thanks I updated the post.

0
Reply
Johnt
January 26, 2024 1:33 pm

Disregard my post. Just needed to fix my DNS entry. Thanks for this awesome
guide!

0
Reply
Dirk
January 28, 2024 12:19 pm

Great tutorial, thank you very much. I only have one problem. My HAOS does not
have an IP. Unfortunately, I can’t access it via the browser. Can I fix this?

0
Reply
Author
Derek Seaman
January 28, 2024 4:52 pm
Reply to  Dirk

You can use the Proxmox console and open a terminal window to HAOS. You can
then use CLI commands to change the IP: https://community.home-assistant.io/t/
how-to-change-ip-adresse-in-cli/332205/2

0
Reply
Jack
February 2, 2024 6:37 am

Fantastic guide. I just migrated HAOS from an old laptop to Proxmox and this
was incredibly easy I am amazed! Thank you!

0
Reply
Bledar
February 10, 2024 8:56 am

Great tutorial. Easy to follow and just concise. Thank you!

0
Reply
Sonic
February 21, 2024 11:42 am

Derek, such a great job on this tutorial, hats off for this gentleman!

Probably the best tutorial on internet, so systematic…

Thank you very much!

0
Reply
Dan
March 9, 2024 12:51 am

Fantastic walkthrough, great content. Keep it up!

0
Reply

  • linkedin
  • threads
  • mastodon
  • x
  • rss

Buy me a coffee

[                    ][                    ]
Search

More results...

Generic filters
[ ]
Exact matches only
[*]
Search in title
[*]
Search in content
[*]
Search in excerpt
Categories

Categories[Select Category                         ]
About

Derek Seaman PhotoDerek Seaman Photo I'm a tech blogger by night, and during
the day I focus on Cyber Security. I hold a number of certifications including
CISSP and VMware VCDX #125. I love smart home technology. [Read More...]
Certified Information Systems Security Professional (CISSP) Certified
Information Systems Security Professional (CISSP)VMware Certified Design Expert
(VCDX) VMware Certified Design Expert (VCDX)

Copyright © 2024 Derek Seaman's Tech Blog

wpDiscuz
[                    ] Insert

  • Facebook
  • Twitter
  • LinkedIn
  • Reddit
  • Email
  • Copy Link

