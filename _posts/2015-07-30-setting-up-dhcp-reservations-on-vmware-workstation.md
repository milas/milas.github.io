---
layout: post
title: Setting Up DHCP Reservations on VMWare Workstation
comments: true
modified:
categories: [howto]
excerpt: A how-to guide for reserving IP addresses for virtual machines in VMWare
tags: [development, vmware, nat, dhcp]
image:
  feature: vmware-howto/feature.png
  thumb: vmware-howto/thumbnail.png
date: 2015-07-30T19:26:57-04:00
---
{% include _toc.html %}

When using multiple VMs for development, I find it easiest to NAT them. Not only does this add an extra layer of security by preventing them from being accessed by other machines on the network beyond the host, it makes it possible to have a consistent configuration across different network configurations.

It's a pain if a machine changes between `10.0.0.123` and `192.168.1.123` based on whether you're at work or home. Even worse is when the DHCP lease expires and it becomes `10.0.0.456`. The NAT configuration in VMWare creates a private subnet just for the VMs, which is the same regardless of the broader network the host machine is connected to.

In this mode, VMWare is acting a the DHCP server, so in its default configuration, we still have the issue of IPs changing when their lease expires (or based on the other you boot the VMs, etc.) With DHCP reservations, the VMs will have sticky IPs; the VMWare DHCP server recognizes each network adapter and gives it a consistent IP.

Let's get started!

**NOTE:** This only applies to the Windows version of VMWare Workstation. I have not tried it on any other edition (e.g. Player). I know for sure that it's different for VMWare Fusion on OS X.

# Configuring the VM Network Adapter
Within the VM guest OS, the network configuration for the VMWare virtual ethernet adatper should be set to automatic/DHCP. This is the default, so unless you customized things, you should be all set!

![Open VM Settings]({{ site.url }}/images/vmware-howto/vm-properties-open.png){: .image-pull-right}
Now, we need to make sure our VM has an adapter associated with our private NAT network. Open the VM settings<sup>[1](#footnote1)</sup>.

![Network Adapter Configuration]({{ site.url }}/images/vmware-howto/vm-properties-network.png)

Switch to the **Network Adapter** section. (If for some reason you don't have a network adapter, click **Add...** to create a new one.) On this screen, make sure that the network connection type is NAT.

![Network Adapter Advanced Configuration]({{ site.url }}/images/vmware-howto/vm-properties-network-advanced.png){: .image-pull-right}
Click **Advanced...** on this screen to open a dialog. This shows the MAC address of the network adapter (if this is a new machine, click **Generate** to create a random one). Copy this somewhere -- you'll need it later.


Click **OK** for both windows to save changes and return to the main VMWare screen. *At this point, if you have any VMs running, either shut them down before continuing.*

# Configuring the NAT Virtual Network
From the **Edit** menu, open the **Virtual Network Editor**.

![Edit -> Virtual Network Editor]({{ site.url }}/images/vmware-howto/virtual-network-editor-open.png)

It's usually pretty slow to load even on a fast machine, so be a bit patient!
{: .notice}

Once it's open, click **Change Settings**. After accepting the UAC/admin dialog, it'll reload.

![Virtual Network Editor]({{ site.url }}/images/vmware-howto/virtual-network-editor-main.png)

In the list of networks, **VMnet8** should be of type NAT. Click on the row to edit its settings. The checkbox for **Connect a host virtual adapter to this network** should be checked. If it's not, check it and click **Apply**.<sup>[2](#footnote2)</sup>

Now, open **DHCP Settings...** for **VMnet8**. 

![DHCP Settings]({{ site.url }}/images/vmware-howto/dhcp-settings.png)

Take note of the range of IP addresses defined by **Starting IP address** and **Ending IP address** (e.g. `192.168.211.128 - 192.168.211.254`). You'll need to pick values in this range to assign to your VMs. Click **OK** for both windows to return to the main VMWare Workstation screen and *quit VMWare entirely before continuing*.

# Adding a DHCP Reservation for the VM
Finally! The actual fun begins, and like most fun things in life, it takes place in a text editor.

Open the file `C:\ProgramData\VMWare\vmnetdhcp.conf`<sup>[3](#footnote3)</sup> in your favorite editor (it's okay if it's Notepad) *as an Administrator*.

{% highlight bash %}
# Virtual ethernet segment 8
# Added at 07/30/15 19:45:17
...
host VMnet8 {
	...
}
# ! ONLY MAKE CHANGES IN THIS REGION !
# End
{% endhighlight %}

Find the **host VMnet8** section. Note the comment delimiters (lines beginning with `#`). All changes you make *must* be between these comments, or they'll get overwritten by VMWare when you least expect it.

Now, for each VM you want to give a reserved IP to, you'll add a section after the **host VMnet8** section but before the `# End` comment.
{% highlight bash %}
host IDENTIFIER {
    hardware ethernet MAC_ADDRESS;
    fixed-address RESERVED_IP;
}
{% endhighlight %}
The identifier must be unique for each VM; make up whatever you'd like. I wasn't brave enough to test with special characters, so I'd just stick to alphanumeric. Fill in the MAC address of the machine you saved earlier and the address you want assigned to the machine (within the allowed range).

Here's a sample complete section:
{% highlight bash %}
# Virtual ethernet segment 8
# Added at 07/30/15 19:45:17
subnet 192.168.211.0 netmask 255.255.255.0 {
range 192.168.211.128 192.168.211.254;            # default allows up to 125 VM's
option broadcast-address 192.168.211.255;
option domain-name-servers 192.168.211.2;
option domain-name "localdomain";
option netbios-name-servers 192.168.211.2;
option routers 192.168.211.2;
default-lease-time 1800;
max-lease-time 7200;
}
host VMnet8 {
    hardware ethernet 00:50:56:C0:00:08;
    fixed-address 192.168.211.1;
    option domain-name-servers 0.0.0.0;
    option domain-name "";
    option routers 0.0.0.0;
}
host Windows10 {
    hardware ethernet 00:50:56:38:AA:C4;
    fixed-address 192.168.211.128;
}
host WindowsXP {
    hardware ethernet 00:50:56:34:C0:1E;
    fixed-address 192.168.211.129;
}
# End
{% endhighlight %}

This is just to give you an idea of what it looks like formatted -- if you just copy and paste this, it's not going to work!
{: .notice}

Save this file once you've added reservations for all the machines you want to have consistent IPs for. If there's a VM you don't care about having a consistent IP, you don't have to include it; VMWare will automatically assign it a non-reserved IP from within the range.

![Windows Services]({{ site.url }}/images/vmware-howto/services.png)

Lastly, you need to restart the VMWare DHCP server. Open **Control Panel -> Administrative Tools -> Services**.

*Alternative Method:* Press Win+R to open the Run dialog, and enter `services.msc`.
{: .notice}

Find the **VMWare DHCP Service** and click **Restart**.

Launch VMWare, start up your VM (if you didn't shut it down before, you'll need to either restart it or force it to renew its DHCP lease), and voila!

# Adding an Entry to the Hosts File (optional)
It can get to be a pain to remember and type in IP addresses over and over again, so I usually create entries for them in my `hosts` file, which lets you map an IP to a host name. For example, I might map `webserver.local` to `192.168.211.128` if that's the IP address of my development web server.

On Windows, the host file is located at `C:\Windows\System32\Drivers\etc\hosts` (notice the lack of file extension). You'll need to edit it with a text editor launched in elevated/admin mode. There are a few examples already in the file, but the format is simple, with each line including an IP address, whitespace, and then the hostname. Anything after a `#` on a line is considered a comment and ignored. For example:

~~~
192.168.211.128		webserver.local		# development web server
~~~

I could then access my webserver by navigating to http://webserver.local/, which is much nicer.


# Footnotes
<a name="footnote1">1</a> If you're creating a VM for the first time, when you get to the last step of the wizard, click **Customize Hardware...** to open the VM settings.

<a name="footnote2">2</a> I've seen an issue on several machines where this option won't stick; in this case, clicking **Restore Defaults** (takes a couple minutes with lots of popups) seems to fix the issue.

<a name="footnote3">3.</a> If you have a funky setup, the `%PROGRAMDATA%` environment variable might help you.