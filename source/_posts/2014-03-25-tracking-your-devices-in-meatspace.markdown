---
layout: post
title: "Tracking your devices in Meatspace"
date: 2014-03-25 19:14:11 +0000
comments: true
categories: spelunking wifi node meteor
---

I work at [uSwitch.com](http://www.uswitch.com) and late last year we moved to some nice new offices.  Unfortunately, whilst awesome, the offices are split over 6 floors, I knew immediately that this would lead to missed connections and endless stair-runs, so I set about building something that would remove that pain.  The end result was this...

![whereabouts-meteor](/images/whereabouts.gif "Whereabouts")

It's loosely called "whereabouts".  This post is about the process of discovery that made it possible to build the two applications that would make it easier for us to find each other throughout the building.  Despite being specific to our network, most of what is detailed here is based on principles that should apply to your own, and it should be possible to achieve the same results.  
A quick shout out to [@randomvariable](https://twitter.com/randomvariable), anytime you see the term "ops", it really means him.  He fixed the network the first time I nuked half of it, and provided insight throughout.

## Network spelunking

As you might expect we have Access Points (APs from here on in) on every floor, each one is providing a wireless network, but is at the same time also wired into the physical network.  It's sensible to assume that all APs would be managed centrally, and this system is usually referred to as a "controller", so I asked the guys in IT Ops for access and logged into the interface to have a poke around.

Peering at the settings in the control panel I noticed there was an option to set a remote [syslog](http://en.wikipedia.org/wiki/Syslog) server.  Knowing that I would be running arbitrary code that would probably be processing logs, it seemed best to make use of this feature.  I went with [RSYSLOG](http://www.rsyslog.com/) and set it up on a fresh Ubuntu instance, which was as simple as doing a `sudo apt-get install rsyslog`.

By tailing the log on the Ubuntu instance I started to see various bits of output related to our network, but nothing particularly useful.  
The "Access Points" tab showed a list of all the APs in the building and their corresponding IP addresses, so I SSHed onto the one that my laptop was connected to, reasoning that it might have some configuration files for me to mess around with.

```sh
$ ssh user@192.168.138.80
user@192.168.138.80s password:

BusyBox v1.11.2 (2013-12-17 16:06:18 PST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

BZ.v3.1.9#
```

From the prompt I could tell that the device is running  [BusyBox](http://www.busybox.net/), a very small, low resource Linux distribution that is often used on embedded systems.  I'm aware of it but have rarely used it.  So I begun looking for config filesin the obvious places.

```sh
BZ.v3.1.9# ls /etc/

aaa2.cfg         board.info       host.conf        iproute2         profile          startup.list
aaa3.cfg         config           hosts            l7-protocols     protocols        sysinit
aaa4.cfg         dropbear         hotplug.d        login.defs       rc.d             sysstat
aaa5.cfg         ethertypes       hotplug2.rules   mime.types       resolv.conf      udhcpc
aaa6.cfg         firewall.user    httpd            modules.d        services         udhcpc_services
aaa7.cfg         fstab            init.d           passwd           shells           version
atheros.conf     group            inittab          persistent       ssl
```

Nothing in particular stood out, so rather than checking them all one at a time, I took a look at running processes.

```sh
BZ.v3.1.9# ps
  PID USER       VSZ STAT COMMAND
    1 deploy    1648 S    init
    ...
  244 deploy    1136 S    /sbin/hotplug2 --persistent --set-rules-file /usr/etc/hotplug2.rules
  246 deploy    1636 S <  /bin/watchdog -t 1 /dev/watchdog
 1087 deploy    1648 S    init
 1088 deploy    1636 S    /usr/bin/klogd -c 8 -n
 1089 deploy    5920 S    /bin/reset-handler
 8439 deploy    3564 S    /bin/hostapd /etc/aaa2.cfg
 8440 deploy    3500 S    /bin/hostapd /etc/aaa3.cfg
 8441 deploy    3512 S    /bin/hostapd /etc/aaa4.cfg
 8443 deploy    3544 S    /bin/hostapd /etc/aaa5.cfg
 8444 deploy    3500 S    /bin/hostapd /etc/aaa6.cfg
 8447 deploy    3504 S    /bin/hostapd /etc/aaa7.cfg
 8561 deploy    1392 S    /sbin/ntpclient -i 86400 -n -s -c 0 -l -h 0.ubnt.pool.ntp.org
 8681 deploy       0 SW   [ubnt-roam-rx-hp]
12484 deploy    1660 S    -sh
14297 deploy    1644 R    ps
```

What's interesting here is [hostapd](http://hostap.epitest.fi/hostapd/) which I recognised from some previous projects involving [Raspberry Pi](http://www.raspberrypi.org/)s.  This daemon is responsible for handling WPA authentication so it seemed like there would be good chance it would know *who* was authenticating.  Thankfully each hostapd process references a configuration file so I opened one of those next.

```
interface=ath6
bridge=br0
driver=atheros
wpa=2
eapol_version=2
wpa_key_mgmt=WPA-EAP
wpa_pairwise=CCMP
ssid=USWRoam
wpa_group_rekey=0
eap_reauth_period=0
ieee8021x=1
auth_server_addr=192.168.143.31
auth_server_port=1812
auth_server_shared_secret=OMITTED!
acct_server_addr=192.168.143.31
acct_server_port=1813
acct_server_shared_secret=ALSO_OMITTED!
logger_syslog=-1
logger_syslog_level=2
```

The last two lines reference syslog, but hostapd documentation is not exhaustive, or if it is I simply don't know how to find it.  A brief google did turn up a [sample config](http://dev.laptop.org/pub/firmware/libertas/thinfirm/hostapd.conf.sample) that contained some comments explaining the values. `logger_syslog` is set to "-1" which logs for all modules so I left it be.  
`logger_syslog_level` however is set to "2" (informational messages), wanting more verbosity, I set it to "1" (debug) in each of the config files and rebooted the device.

The tailed output from the Ubuntu instance flicked up some messages about the AP going down and coming back up, but nothing more detailed than before.  Upon closer inspection I noticed that the config files had reset to their original states.  Assuming this was probably by design, I started wandering around the more common places people drop their shell aliases.  In `/etc/profile` I found an amusing comment, and a helpful looking command.

```sh
# habits from ubuntu...
alias sudo=''
alias save='cfgmtd -w -p /etc/'
```

I guess Ubiquiti engineers really like Ubuntu.  Anyhow... the `save` alias speaks of `cfgmtd`, so I checked help to see what I could learn.

```sh
BZ.v3.1.9# cfgmtd -help
Usage: cfgmtd [options]
	-t <type>			- Configuration type to use [1(active)|2(backup)]. (Default: 1(active))
	-f <config file>		- Configuration file to use. (Default: /tmp/system.cfg)
	-p <persistent directory>	- Directory to persistent dir. (Default: none)
	-w				- Write to flash action.
	-r				- Read from flash action.
	-o <mtd|file name>		- Use mtd or file name. (Default: cfg)
	-h				- This message.
```

From the flags I could see it was used for reading and writing config to the flash memory of the device, which persists through reboots.  It also told me that the default config is in `/tmp/system.cfg`, **all 480 lines** of it.

```
aaa.2.br.devname=br0
aaa.2.devname=ath1
aaa.2.driver=madwifi
aaa.2.eapol_version=2
aaa.2.radius.acct.1.ip=192.168.143.31
aaa.2.radius.acct.1.secret=OMITTED!
aaa.2.radius.auth.1.ip=192.168.143.31
aaa.2.radius.auth.1.port=1812
aaa.2.radius.auth.1.secret=ALSO_OMITTED!
aaa.2.ssid=Notcutt House
aaa.2.status=enabled
aaa.2.verbose=3
aaa.2.wpa.1.pairwise=TKIP CCMP
aaa.2.wpa.group_rekey=0
aaa.2.wpa.key.1.mgmt=WPA-EAP
aaa.2.wpa.psk=MOAR_OMISSION!
aaa.2.wpa=3
```

Line 12 in the abbreviated sample showed the verbosity being set to 3, which would suggest that only "notifications" are being logged, but when I checked the config it distinctly showed that `logger_syslog_level` is set to 2 for informational messages.  At first I didn't notice this mismatch, but eventually through repetition I caught the problem and set the correct value.  With that settled I flashed the device once more.

```sh
cfgmtd -f /usr/etc/system.cfg -w && reboot
```

Finally the tailed output from syslog started to show some users authenticating.  The IP address is for the particular AP that received the event, the [MAC Address](http://en.wikipedia.org/wiki/MAC_address) refers to the device connecting, in this case my laptop.

```
Dec 15 18:33:03 192.168.138.102 hostapd: ath6: STA d0:df:9a:74:e3:cd IEEE 802.1X: STA identity 'USWITCH\baris.balic'
```

I thought it would be plain sailing from here on out, but of course that was not the case.  Over the next day or so "identity" messages stopped appearing in the logs.  The configs for the APs were being reset. I was fairly sure it would be triggered by the controller as it was affecting all of the APs.
Through trial and error I discovered that certain changes on the controller would reprovision the APs.

SSHing into the controller, I had no idea where to look, so I resorted to reviewing the running processes like before and found some interesting candidates for inspection.

```sh
21196 ?        Ss     0:00 unifi -home /usr/lib/jvm/java-6-openjdk-amd64 -cp /usr/share/java/commons-daemon.jar:/usr/lib/unifi/lib/ace.jar -pidfile /var/run/unifi/unifi.pid -procname unifi -outfile SYSLOG -errfile SYSLOG -Djava.awt.headless=true -Xmx1024M com.ubnt.ace.Launcher start
21198 ?        S      0:05 unifi -home /usr/lib/jvm/java-6-openjdk-amd64 -cp /usr/share/java/commons-daemon.jar:/usr/lib/unifi/lib/ace.jar -pidfile /var/run/unifi/unifi.pid -procname unifi -outfile SYSLOG -errfile SYSLOG -Djava.awt.headless=true -Xmx1024M com.ubnt.ace.Launcher start
21199 ?        Sl    31:56 unifi -home /usr/lib/jvm/java-6-openjdk-amd64 -cp /usr/share/java/commons-daemon.jar:/usr/lib/unifi/lib/ace.jar -pidfile /var/run/unifi/unifi.pid -procname unifi -outfile SYSLOG -errfile SYSLOG -Djava.awt.headless=true -Xmx1024M com.ubnt.ace.Launcher start
21218 ?        Sl   803:13 /usr/lib/jvm/java-6-openjdk-amd64/jre/bin/java -Xmx1024M -jar /usr/lib/unifi/lib/ace.jar start
21235 ?        Sl   911:26 bin/mongod --dbpath /usr/lib/unifi/data/db --port 27117 --logappend --logpath logs/mongod.log --rest
```

Seeing the [mongoDB](https://www.mongodb.org/) process, I connected to it with `mongo --port 27117`, which gave me access to the "ace" database/collections.  It's mostly stats and settings from the control panel on the controller.

The remaining Java processes referenced "unifi" paths on the filesystem, which is the name of the product itself.  So I pulled down the "ace.jar" mentioned in the processes and fired up [JD-GUI](http://jd.benow.ca/), a tool that takes [Java bytecode](http://en.wikipedia.org/wiki/Java_bytecode) and spits out source code.  
Most of the time the result looks like shit, symbol names are not preserved and any meaningful structure is lost.  If you're feeling brave, I created [this gist](https://gist.github.com/barisbalic/e3dd21cbaa66b25a6e96) for anyone that wants to see what I mean.  This was a class called "String", which contained the code for reading and writing settings, I found it by doing a global search for "aaa" which was part of the filename for the wifi configs.  
From that massive file the only interesting part is this, as the values are hardcoded it's kind of fortunate that one would provide enough verbosity.

```java
"verbose", D.Òo0000() ? "3" : "2"
```

The method `Òo0000` on the class `D`, was potentially calling another two methods on other classes, if either case were to return the string "debug" then the desired `logger_syslog_level` value will be used whenever the controller reprovisions.

```java
public static boolean Òo0000()
{
  return ("debug".equals(õO0000.getString("build.type"))) || (øO0000.o00000("debug", false));
}
```

I'll spare you the pain of looking at obfuscated code and skip to the conclusion: I eventually discovered the "build.type" mentioned above would be read from a file called "product.properties", and that the "debug" would be read from "system.properties".  Both of which appeared in the compiled JAR itself.  Luckily though, the code actually attempted to read from the file `/usr/lib/unifi/data/system.properties` before resorting the internal version.  By adding `debug=true` to it I was able to override the config.  I was finally ready to start building something.

## Building Whereabouts

Whereabouts is comprised of two pieces: the first is the backend that tails the syslog and upserts documents in a mongo collection, and the second is a basic [meteor](https://www.meteor.com/) app that tells you who is where in the building.  
They are called [whereabouts-syslog-tail](https://github.com/barisbalic/whereabouts-syslog-tail) and [whereabouts-meteor](https://github.com/barisbalic/whereabouts-meteor) respectively.  They were hacked out quickly so please get in touch, create an issue or send a pull request if you need some help or want to contribute.

### whereabouts-syslog-tail

When the first set of "identity" messages started appearing I set up a small [Node.js](http://nodejs.org/) script to tail the syslog and write entries into a mongo collection, at first this was just comprised of the username and a MAC Address.  Watching for connection and disconnection messages seemed to cover only edge cases: the collection didn't reflect the state of the network, probably because it was missing things like association and deauthentication.  
I googled for hostapd documentation but again couldn't find anything useful, so this time I decided to checkout the code.  A spate of "event X notification" messages would appear in the log, so I looked for the string in the source code.

```cpp
int wpa_auth_sm_event(struct wpa_state_machine *sm, wpa_event event)
{
  int remove_ptk = 1;

  if (sm == NULL)
    return -1;

  wpa_auth_vlogger(sm->wpa_auth, sm->addr, LOGGER_DEBUG,
       "event %d notification", event);

  ...
}
```

The shortened example above shows a `wpa_event` being passed to the logger function, the event turned out to be a basic enum.

```cpp
typedef enum {
  WPA_AUTH, WPA_ASSOC, WPA_DISASSOC, WPA_DEAUTH, WPA_REAUTH,
  WPA_REAUTH_EAPOL, WPA_ASSOC_FT
} wpa_event;
```

Each of these seven events is implies whether a device is connected or not, for example, seeing a `WPA_DEAUTH` is enough to say a device is disconnected until another message indicates otherwise.  I modified the script to process these events and update the collection, adding a human-friendly name for each location instead of an IP address.

There's a little more tidy-up done in the app, and it exposes a very basic API, but you can refer to the [repository](https://github.com/barisbalic/whereabouts-syslog-tail) for more detail.

### whereabouts-meteor

Meteor is particularly good for rapidly building single page apps, out of the box it gives you:

- Hot code deploys.
- Live updating HTML.
- Handy data synchronisation.

The live updates mean that data is bound to templates, when the data changes, the templates react to those changes.  This saves you from having to wire up various events and write other boiler-plate code.

The data synchronisation is also pretty awesome, by default a Meteor app will use a client-side mongo implementation called [minimongo](https://www.npmjs.org/package/minimongo).  It wiring up callbacks for any changes to documents, these changes will then get pushed to all connected clients at the same time.  
When you specify a real mongodb instance with the `MONGO_URL` environment variable, Meteor will use that instead, keeping the synchronisation feature.

Given that the the mongo collection reflects the state of the network at any given time, the Meteor app took only a few minutes to create and simply finds all known devices.  They are ordered by whether they are connected, followed by their last connection time.  It's a bit basic but it serves the purpose and acts as a reasonably straight-forward example, I will happily link to any other interfaces that are written.

## Further development?

An issue that arose from tracking devices is that people often have more than one.  The next obvious feature is allowing people to name their devices by visiting the application with said devices.  It's not possible to obtain a client MAC address in a web-app without resorting to something horrible like [ActiveX](http://en.wikipedia.org/wiki/ActiveX) controls, so instead the app will need to compare the client IP address with those of known devices to find a match.
The IT Ops guys were happy to push our [DHCP](http://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) logs to the remote syslog server I set up, and [whereabouts-syslog-tail](https://github.com/barisbalic/whereabouts-syslog-tail) has the code necessary to associate an IP address with a device, but work still needs to be carried out on the front end.

Aside from that I would be happy to work on making the whole solution more generic, and thus more suitable for other networks, but in order for that to happen I'll need volunteers to give feedback on their setup and what does/doesn't work for them, so please get in touch if you'd like to/some help.