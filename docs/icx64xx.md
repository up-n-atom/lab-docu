# Setting up Brocade ICX6450

## Preparation 
Download the ZIP below, which contains the firmware files and documentation you'll need.  

[```Brocade v8030 Firmware/Docu Zip```](http://fohdeesha.com/data/other/brocade/8030t.zip)  
```SW version: 08030t```  
```ZIP Updated: 03-21-2019```  
```MD5: 6f4b1c6b2f1faf4c8c5d1f647b6505f2```  


**Connect** to the switches serial/console port on the front using a program like Putty (9600 8N1), and connect any of the normal switch ports to your network (do NOT use the dedicated management port).

You need to set up a temporary TFTP server - I recommend [Tftpd32 Portable Edition](http://www.tftpd64.com/tftpd32_download.html) if you're on Windows and don't want to install anything. Point the server to an empty folder to serve files from. From the ZIP, copy the bootloader from the ```Boot``` folder into your tftp server directory. Then, from the ```Images``` folder, copy over the OS image to the same place. If you have a PoE model, copy over the PoE firmware from the `PoE Firmware` folder to your TFTP directory as well.  

Power on the switch while watching your serial terminal - start smashing the `b` key until you're dropped into the bootloader prompt, which looks like `ICX64XX-boot>>` . If you missed the prompt and it boots the OS instead, pull power and try again.  

Now at the boot prompt, we tell the switch to clear all current configs and old keys, so it boots into a fresh state:

```
factory set-default
```
To confirm this action, you must send CAPITAL `Y` - sending a lowercase `y` will just make it abort.   

Now just tell the switch to reboot:

```
reset
```
It will boot into the full OS and you can continue to the next section.  

**Note:** If you get an error stating `factory set-default` is not a valid command, this means your switch has a very old bootloader. To clear the existing passworded config in this case, do the following:
```
#only follow this section if factory set-default did not work
no password
boot
#it will boot the OS. Once booted:
enable  
erase startup-config  
reload
#it will boot fresh with a clear config, then you can move to the next section
```

**Note:** If your switch says that it is part of a stack even after doing the above, then you will need to unstack it before being able to configure and update it. You'll see several prompts when booting that it is a stack member and the command menu will show stack specific commands (ex: stack, simulate-non-stacking-unit). You can unstack your switch by running the following command:
```
stack unconfigure clean
# hit y at the prompt
```
You can then continue on to the below.

## Initial Configuration & update

Now that it's booted into the full OS you may get ***TFTP timed out*** errors in the console, this is normal. just hit enter until they go away. We'll fix that in the next section. Now to make any changes we must enter the enable level:
```
enable
```
Now we enter the configure terminal level to make config changes:
```
configure terminal
```
Now we turn off the DHCP client, so it doesn't automatically grab an IP and look for a TFTP config (the cause of the earlier timeout messages):
```
ip dhcp-client disable
write memory
exit
```
>**Note:** If during the `ip dhcp-client disable` command you get an invalid input error, your switch probably came with the layer 2 only firmware loaded. In that case, just run `ip address 192.168.1.55/24` (replace the IP) - this will give it a temporary IP, so it can load the layer 3 firmware. Then skip down to the `Load The New Images` section below, and use those commands to copy in the latest layer 3 firmware and reload. Once the switch comes back up, **don't forget** to come back and follow this **Initial Configuration** section all the way through now that you have the proper firmware!

Now just reload the switch so it comes back up without an IP assigned to a port via DHCP:

```
reload
```

Once it's back up, enter the configure level again:

```
enable
configure terminal
```
We need to give it an IP so we can load the new firmware. By default, all ports are in VLAN 1, so it will behave like a typical switch. First we need to give VLAN 1 its own virtual interface:
```
vlan 1
router-interface ve 1
exit
```
Now we need to assign that virtual interface an address. Choose an IP that is unused in your subnet, and out of your DHCP server range (ping it first to be sure it's unused):
```
interface ve 1
ip address 192.168.1.55/24
exit
write mem
```

## Load The New Images

Now that the switch has an IP address, we can TFTP in the new images, then reload:
```
exit
copy tftp flash 192.168.1.8 kxz10105.bin bootrom  
copy tftp flash 192.168.1.8 ICX64R08030t.bin primary
reload
```
>Note: if you skipped to this section because your switch had layer 2 firmware on it and a note in the previous section instructed you to, go back above and go through the whole **Initial Configuration & update** section once the switch finishes rebooting.  

The switch will reload, loading the new software. Continue on to the **Configuration Details** section below.

## Configuration Details

Your switch should now be freshly booted with the latest layer 3 firmware image and bootloader. First give the switch a name:

```
enable
configure terminal
hostname intertubes
```
Now tell it to generate an RSA keypair - this is the first step to enable SSH access:
```
crypto key generate rsa
```


## Update PoE Firmware
 If your switch is the PoE model, you need to update the PoE controller firmware. If it's a non-PoE model, skip this step. Assuming you completed the previous section and the switch now has in-band network access, just do the following:

**Note:** if you have the small C12 version of this switch, use the `icx64xxc12_poeplus_02.03.09.fw` file instead.
```
exit
inline power install-firmware stack-unit 1 tftp 192.168.1.8 icx64xx_poeplus_02.1.0.b004.fw
#after a few seconds, hit enter to return to cli
#save changes you made from the previous section
write memory
#reload the switch
reload
#you'll probably get a message that it hasn't finished. it can take up to 10 minutes
#run "show log" occasionally to monitor the update progress
#try the "reload" command again once it's reached 100%
#once you've rebooted back into the OS:
enable
configure terminal
```
Now you have the latest PoE firmware, and can continue on.

## If Access Protection Is NOT Required
If you do **not** want to password protect access to the switch (you're using it in a lab), follow this section. If you'd like to password protect it, skip to the next section.

Allow SSH login with no passwords configured:
```
ip ssh permit-empty-passwd yes
```

## If Access Protection IS Required (or WEB-UI Access)
If you do want to secure access to the switch, or use the (limited) web UI, follow this section. If not, skip it.  

To secure the switch, we need to create an account - "root" can be any username string you wish:
```
username root password yourpasshere
```
We also need to tell it to use our new local user account(s) to authorize attempts to log in, use the webpage, as well as attempts to enter the ```enable``` CLI level:
```
aaa authentication login default local
aaa authentication enable default local
aaa authentication web default local
```
If you wanted to use the WEB UI, you can now log into it using the credentials you created above.  

You should enable authentication for telnet access as well:
```
enable telnet authentication
```
If your switch is outside of your home, or accessible by others in any way, telnet should be disabled entirely, and access to the serial console should also be password protected. Otherwise skip this step at your discretion:

```
no telnet server
enable aaa console
```
### OPTIONAL: Key Based SSH Access
If you have followed the above to set up authentication, and also wish to disable password-based SSH login and set up a key pair instead, follow this section. If not, skip it. Enable key login, and disable password login:
```
ip ssh key-authentication yes
ip ssh password-authentication no
```
Now we have to generate our key pair with [puttygen](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) on windows or ```ssh-keygen -t rsa``` on linux. The default settings of RSA @ 2048 bits works without issue. Generate the pair and save out both the public and private key. 

Copy the public key file to your TFTP server. Then use the following command to import it into your switch:
```
ip ssh pub-key-file tftp 192.168.1.49 public.key
```
You shouldn't need to be told basic key management if you're following this section, but just in case - copy your private key to the proper location on the *nix machine you'll be SSH'ing from, or if you're on windows, load it using [pageant](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html). Now when you SSH to the switch, it will authenticate using your private key.

## Saving & Conclusions
Whenever you make changes (like above) they take effect immediately, however they are not saved to onboard flash. So if you reboot the switch, they will be lost. To permanently save them to onboard flash, use the following command:
```
write memory
```
Your switch now has a basic configuration, as well as an IP address you can telnet or SSH to for further configuration.  

Some more useful general commands:  

Show chassis information like fan and temperature status:
```
show chassis
```

Show a table of all interfaces:
```
show interface brief
```
To show one interface in detail:
```
show interfaces ethernet 1/1/1
#Also works for virtual interfaces:
show interfaces ve 1
```
Give a port a friendly name:
```
interface ethernet 1/1/1
port-name freenas
show interfaces brief ethernet 1/1/1
exit
```
Show the running configuration:
```
show run
```
Show the system log:
```
show log
```

To remove configuration options, put a ```no``` in front of them at the appropriate CLI level:
```
no hostname intertubes
```
## Tips
To exit the CLI level you are at, use `exit`. So assuming you are still at the ```configure terminal``` level, type the following to exit back to the ```enable``` level:
```
exit
```
Commands can also be shortened, as long as they are still unique. So to re-enter the configure terminal level, Instead of typing the entirety of ```configure terminal```, the following will also work:
```
conf t
```
There is also tab help and completion. To see all the commands available at the current CLI level, just hit tab. To see the options available for a certain command, just type that command (like ```ip```) then hit tab a couple times.

## Advanced Configuration

### Default Route & DNS
To give the switch a default route and a DNS server so it can reach external hostnames and IP's (to ping external servers or to update time via NTP etc), do the following. replace the IP with the IP of your gateway/router/etc. Assuming you are still at the ```configure terminal``` level:

```
ip dns server-address 192.168.1.1
ip route 0.0.0.0/0 192.168.1.1
```
### NTP
To have the switch keep its time synced via NTP (so its logs make more sense), use the following. If you live in an area that doesn't use Daylight Savings, skip the ```clock summer-time``` command. Use tab completion for the timezone command to see what's available. The IP's in the following example are google's NTP servers and work well for most cases:
```
clock summer-time
clock timezone gmt GMT-05
ntp
disable serve
server 216.239.35.0
server 216.239.35.4
exit
```
### SNMP

To quickly enable SNMPv2 (read only), follow the below. SNMP v3 is available but you'll have to refer to the included documentation:
```
snmp-server community public ro
```

### Saving
If you made any of the above extra changes, remember they have not been saved to onboard flash yet. Do so:
```
write memory
```

## SFP/Optics Information
Brocade does not restrict the use of optics by manufacturer, they'll take anything given it's the right protocol. However optical monitoring information is disabled unless it sees Brocade or Foundry optics.  

So if you want to see information like this :

```
telnet@Route2(config)#sh optic 5
 Port  Temperature   Tx Power     Rx Power       Tx Bias Current
+----+-----------+--------------+--------------+---------------+
5       32.7460 C  -002.6688 dBm -002.8091 dBm    5.472 mA
        Normal      Normal        Normal         Normal
```
You'll need to pick up some official Brocade or Foundry optics on ebay, or buy some flashed optics from FiberStore.  

### Thanks:
[**Fohdeesha**](http://fohdeesha.com/)  


### Contributing:
The markdown source for these guides is hosted on [**my Github repo.**](https://github.com/Fohdeesha/lab-docu) If you have any suggested changes or additions feel free to submit a pull request.  

```Documentation version:``` [ v1.8 (03-06-19)](https://github.com/Fohdeesha/lab-docu/commits/master) 
