#Building a DNS Blackhole with FreeBSD

Retrieved on 2014-07-25 17:13:34 UTC+0000 from http://www.pintumbler.org/Code/dnsbl

##Preamble

This document will outline how to setup FreeBSD to act as a DNS Blackhole (DNSBH).

- Hardware requirements
- How to configure BIND
- How to automate the update process
- How to setup Apache to provide an information page to the end user
- Monitoring examples and ideas
- Summary

###What is a DNS Blackhole and why would I want one?

A DNS blackhole (DNSBH) in its simplest form is just a box running bind that maintains a listing of malicious domains. When clients request a 'flagged' domain they will be redirected to either themselves (localhost), or to a safe local location that explains to the user why they just ended up where they did.

Keep in mind that this isn't an inline device and depending of the deployment strategy can be easily bypassed. That being said, It is low cost, low maintenance, somewhat flexible and if deployed correctly can still be quite effective at threat mitigation.

The real truth is in the numbers though. Our deployment blocks on average 1000 requests/day. Even if only 1% of that would have led to an infection, the hardware and operation costs of the device would be recouped in a couple weeks.

##Hardware Requirements

The beauty of this solution is that it requires very little horsepower. My first incarnation consisted of Dell Optiplex desktop. This bad boy had 1GB of RAM and a dual core Pentium 4 processor@3.00GHz complete with an 80GB SATA drive. This served 5000 clients without any issues.

The second incarnation I spec'd up a bit because I was interested in running IDS components on the inbound requests to see if there was added value to this approach. The incumbent is a 2U server with 2 dual core processors@2.8GHz and 5GB of RAM. It also has RAID. This is overkill of course but it opens up some data mining possibilities.

##How to configure BIND

Most everything you need is already available on a base install. I usually start with a custom install, use the Kern-Developer distribution (somewhat minimalistic) and also grab the ports tree.

Once the box is up:

1) Edit `/etc/rc.conf` and make sure it has (add where necessary and substitute your interface name, gateway, etc.) The IP address of this server will be 10.0.0.1:

```
hostname="bhdns.mydomain.ca"
ifconfig_bge0="inet 10.0.0.1&nbsp; netmask 255.255.255.0"
defaultrouter="10.0.0.254"
named_enable="YES"
named_program="/usr/sbin/named"
named_pidfile="/var/run/named/pid"
named_uid="bind"
named_chrootdir="/var/named"
named_chroot_autoupdate="YES"
named_symlink_enable="YES"
```

Save and exit the file.

2) Edit `/etc/hosts`:

```
::1        localhost localhost.my.domain bhdns
127.0.0.1  localhost localhost.my.domain bhdns
10.0.0.1   bhdns.mydomain.ca bhdns
```

Save and exit the file.

3) Edit `/etc/resolv.conf` and add:

```
nameserver 127.0.0.1
domain mydomain.ca
nameserver [upstream provider]
```

Save and exit the file.

4) Now take a look at `/etc/namedb`. The file is well documented. These are the changes/additions that you should make:

In the options section you need to add an entry to allow clients access. This fictitious install is on a 10 network so it would look like:

```
allow-query { 10.0.0.0/8; };
```

Next you want to set the address that the service will listen on. Use the same address you set in `rc.conf`:

```
listen-on { 10.0.0.1; };
```

Now you can set up a forwarder, in my case the same one used in `/etc/resolv.conf`:

```
forwarders { [upstream provider]; };
```


This isn't a requirement but by using a forwarder you will take advantage of a more local cache which will increase performance.

The rest of the settings you can gloss over (unless you want to do more than just DNSBH). At the very end of the file though, you want to add the actual block lists. You can have as many as you wish but to keep things simple I will focus on a single list from a single source. I will use the list from http://www.malwaredomains.com/files/spywaredomains.zones as an 
example:

So:

```
include "/etc/namedb/blackhole/spywaredomains.zones";
```

Save and exit the file.

5) Create a folder called blackhole (same location you specified above) and fetch the zonefile:

```
~# mkdir /etc/namedb/blackhole
~# cd /etc/namedb/blackhole
~# fetch http://www.malwaredomains.com/files/spywaredomains.zones
```

This file contains entries that look like:

```
zone "razdrochi.ru" {type master; file "/etc/namedb/blockeddomain.hosts";};
```

With this loaded, any client request for `razdrochi.ru` will be redirected to whatever we have set up in `/etc/namedb/blockeddomain.hosts`. Essentially, all we are doing is mapping all of the domains listed in that file to the same DNS (A) record.

Lets create this record now.

6) Edit `/etc/namedb/blockeddomain.hosts`:

```
; This zone will redirect all requests back to the blackhole itself.

$TTL    86400   ; one day

@       IN      SOA    bhdns.mydomain.ca. bhdns.mydomain.ca. (
                       1
                       28800   ; refresh  8 hours
                       7200    ; retry    2 hours
                       864000  ; expire   10 days
                       86400 ) ; min ttl  1 day
                NS     bhdns.mydomain.ca.
                A      10.0.0.1
*       IN      A      10.0.0.1
```

Save and exit the file.

Note: You can redirect the request to anywhere you wish but it is worthwhile to send the user to a place that explains what just happened. If not, the user might get confused and open a vague "The Interweb is broken" helpdesk ticket.

We should be good to go. Try and start the service:

```
~# /etc/rc.d/named start
```

For debugging (or other) reasons it might be worth it to separate named logs from the syslog catchall. To do this, edit `/etc/syslogd.conf` and add:

```
!named
*.*    /var/log/named.log
```

Save and exit the file and then:

```
~# touch /var/log/named.log
~# /etc/rc.d/syslogd restart
```

##How to automate the update process

To make things a little easier, I made a simple script to manage the update process. The script does the following:

1. Fetches the zonefile
2. Performs a comparison with the current file, if there are no changes, exit. If there are then
3. Make note of the additions/removals
4. Put the new zonefile in place
5. Restart the service
6. Email the changes to an admin

The script doesn't accommodate multiple files. If you will be using numerous sources just concatenate them into a single file. The script of course will need to be modified if you intend to run it on a system other than FreeBSD.

You can grab the script from http://www.pintumbler.org/getzones.sh or just follow the steps below.

1) Download getzones.sh:

```
~# cd /etc/namedb/blackhole
~# fetch http://www.pintumbler.org/getzones.sh
~# chmod +x /etc/namedb/blackhole/getzones.sh
```

2) Create the temp directory:

```
~# mkdir /etc/namedb/blackhole/work
```

3) Add an entry to roots crontab to run the script daily:

```
~# crontab -e
```

once the editor comes up, input the following line:


```
0    0       *       *       *       /etc/namedb/blackhole/getzones.sh > /dev/null 2>&1
```

Save and exit the file.

This will update the file every day at midnight.

My bash is pretty shoddy so you might want to test it out first :)

```
~# /etc/namedb/blackhole/getzones.sh
```

If it doesn't work, send me an email and I will try and help you out.

##How to setup Apache to provide an information page

As I mentioned earlier, to avoid confusion it is a good idea to send the users to an information page. You can use any web server here, Apache is way overkill but its what I know.

1) Install Apache. You can do this however you wish, I will just use the ports tree:

```
~# cd /usr/ports/www/apache22; make install clean
```

2) Edit `/usr/local/etc/apache22/httpd.conf` and make the following changes (in order of appearance):

```
Listen 10.0.0.1:80
ServerAdmin atech@mydomain.ca
ServerName 10.0.0.1:80
DocumentRoot "/usr/local/www/dnsbh"
ErrorDocument 500 /404.html
ErrorDocument 404 /404.html
ErrorDocument 402 /404.html
```

The rest of the defaults are fine.

3) Create the web directory:

```
~# mkdir /usr/local/www/dnsbh
```

4) Create `/usr/local/www/dnsbh/index.html` this will be the main landing page when folks are redirected:

```
<!DOCTYPE html>
<html>
<head>
<title>Your Org Name - IT services</title>
<script type="text/javascript">url = parent.window.location.href;</script>
</head>
<body> 
<h3>Security Notice...</h3>

You have been redirected to this page because the website that you tried to visit has been known to harbor 
or distribute Spyware, Viruses or other forms of malicious software. 

<br><br> 

As part of our Information Security Policy we maintain a listing of potentially harmful sites to assist in the protection and stability of our computing resources. This is also done to protect users from divulging personal information to third parties where it could be used for illicit purposes such as Spam or Fraud. 

<br><br> 

If you feel your access to this web site is a requirement, contact your local Information Technology Services department for assistance. 

</body> 
</html> 
```
Save and exit the file.

5) Create `/usr/local/www/dnsbh/warn.png`. This image should be around 127 X 57 pixels and contain something that identifies your organization along with something that conveys 'warning' either through words or images.

What happens is a lot of ad space can point to blocked domains and while the core site loads OK, you end up with your warning index document trying to pack into a small ad space. There might be a better way around this but the simplest solution I found is to use a 404 document which displays the image and that is linked back to the main page.

It is enough to prod the user along.

6) Create `/usr/local/www/dnsbh/404.html`. This should look something like:

```
<!DOCTYPE html>
<html>
<head>
<body>
<a href="/index.html" target="_new"><img border="0" src="/warn.png"></a>
</body>
</html>
```

That's it. There is obviously a lot of options here, this should be enough to get you started.

##Monitoring examples and ideas

I have some stuff set up that leverages our existing Netflow data to try and add a little value to what goes on with this box.

The examples below are simply trending connections to port 80.

The shortcomings of this solution aside, that second image is quite compelling. You don't even need an analyst to interpret it. This is the kind of stuff that can be easily offloaded because the message is so poignant.

This example shows a typical day of activity:

<a href="http://www.pintumbler.org/Code/dnsbl/dnsbh1.png?attredirects=0" imageanchor="1"><img border="0" src="http://www.pintumbler.org/_/rsrc/1303426214049/Code/dnsbl/dnsbh1.png"></a>

This shows an infection:

<a href="http://www.pintumbler.org/Code/dnsbl/dnsbh2.png?attredirects=0" imageanchor="1"><img border="0" src="http://www.pintumbler.org/_/rsrc/1303426235351/Code/dnsbl/dnsbh2.png"></a>

A nice summary table like the one below is very useful. It is important to not just focus on hits but just how much data a client trying to send out. Large payloads repeatedly sent to this device should be an instant alarm.

<a href="http://www.pintumbler.org/Code/dnsbl/dnsbh3.png?attredirects=0" imageanchor="1"><img border="0" src="http://www.pintumbler.org/_/rsrc/1303426262027/Code/dnsbl/dnsbh3.png"></a>

Keep in mind too that that this is a webserver people are connecting to. Which means: `LOGS!` Aside from the fact that we can use these to clarify events, the data is screaming to be mined. I am not quite there yet, but soon.

###Summary

Being a little short handed at work, this was one of the first security solutions that I gravitated towards. The passiveness (and price) just made sense. A DNSBH can dramatically improve an organizations overall security posture for next to nothing. Yes, understand that It is NOT going to help you deal with intelligent threats, but you know what? that doesn't really matter because most aren't.

The sludge, the unworthy.. that is where this solution shines. If you are short on time, and short on resources, this is a gift.

Thanks to everyone that has contributed to and maintained these lists over the years. You have made my life a lot easier :)
