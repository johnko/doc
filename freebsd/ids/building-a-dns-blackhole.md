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

<div id="sites-canvas-main-content">
<table xmlns="http://www.w3.org/1999/xhtml" cellspacing="0" class="sites-layout-name-one-column sites-layout-hbox"><tbody><tr><td class="sites-layout-tile sites-tile-name-content-1"><div dir="ltr">
<br>
<font size="4"><b><br>
<font size="5"><br>
<font size="4"><a name="l2">How to configure BIND</a></font></font></b></font><font size="5"><b><br>
</b>
</font><br>
Most everything you need is already available on a base install. I usually start with a custom install, use the Kern-Developer distribution (somewhat minimalistic) and also grab the ports tree.<br>
<br>
Once the box is up:<br>
<br>
1) Edit <i><b style="color:rgb(0,0,0)">/etc/rc.conf</b></i> and make sure it has (add where necessary and substitute your interface name, gateway, etc.) T<b>he IP address of this server will be 10.0.0.1</b>:<br>
<br style="font-family:arial,sans-serif">
<i><b style="font-family:arial,sans-serif">hostname="bhdns.mydomain.ca"<br>
ifconfig_bge0="inet 10.0.0.1&nbsp; netmask 255.255.255.0"<br>
defaultrouter="10.0.0.254"<br>
named_enable="YES"<br>
named_program="/usr/sbin/named"<br>
named_pidfile="/var/run/named/pid"<br>
named_uid="bind"<br>
named_chrootdir="/var/named"<br>
named_chroot_autoupdate="YES"<br>
named_symlink_enable="YES"</b></i><br>
<br>
Save and exit the file.<br>
<br>
2) Edit<b style="color:rgb(153,0,0)"> </b><b><span style="color:rgb(153,0,0)"><i><span style="color:rgb(0,0,0)">/etc/hosts</span></i></span></b>:<br>
<br>
<i><b style="font-family:arial,sans-serif">::1<span><span>&nbsp;&nbsp; &nbsp;<span>&nbsp;&nbsp; &nbsp;</span></span></span><span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; </span>localhost localhost.my.domain bhdns<br>
127.0.0.1<span><span>&nbsp;&nbsp; &nbsp;</span></span>localhost localhost.my.domain bhdns<br>
10.0.0.1<span>&nbsp;&nbsp; &nbsp;</span><span>&nbsp; </span>bhdns.mydomain.ca bhdns</b><br style="font-family:arial,sans-serif">
</i><br>
Save and exit the file.<br>
<br>
3) Edit<b><span style="color:rgb(153,0,0)"> </span><span style="color:rgb(153,0,0)"><i><span style="color:rgb(0,0,0)">/etc/resolv.conf</span></i></span></b> and add:<br>
<br>
<i><b>nameserver 127.0.0.1</b></i><br>
<i><b>domain mydomain.ca<br>
nameserver &lt;upstream provider&gt;</b><br>
</i><br>
Save and exit the file.<br>
<br>
4) Now take a look at <b><span style="color:rgb(0,0,0)">/etc/namedb</span></b>. The file is well documented. These are the changes/additions that you should make:<br>
<br>
In the options section you need to add an entry to allow clients access. This fictitious install is on a 10 network so it would look like:<br>
<br>
<i><b>allow-query { 10.0.0.0/8; };</b><br>
</i><br>
Next you want to set the address that the service will listen on. Use the same address you set in rc.conf:<br>
<b><br>
<i>listen-on { 10.0.0.1; };</i></b><br>
<br>
Now you can set up a forwarder, in my case the same one used in /etc/resolv.conf:<br>
<i><b><br>
forwarders { &lt;upstream provider&gt;; };</b></i><br>
<br>
This isn't a requirement but by using a forwarder you will take advantage of a more local cache which will increase performance.<br>
<br>
The rest of the settings you can gloss over (unless you want to do more than just DNSBH). At the very end of the file though, you want to add the actual block lists. You can have as many as you wish but to keep things simple I will focus 
on a single list from a single source. I will use the list from <a href="http://www.malwaredomains.com/files/spywaredomains.zones" target="_blank">malwaredomains</a> as an 
example: <br>
<br>
So:<br>
<br>
<i><b>include "/etc/namedb/blackhole/spywaredomains.zones";</b></i><br>
<br>
Save and exit the file.<br>
<br>
5) Create a folder called blackhole (same location you specified above) and fetch the zonefile:<br>
<br>
<i><b>~# mkdir /etc/namedb/blackhole<br>
~# cd /etc/namedb/blackhole<br>
~# fetch http://www.malwaredomains.com/files/spywaredomains.zones</b></i><br>
<br>
This file contains entries that look like:<br>
<br>
<i><b>zone "razdrochi.ru"&nbsp; {type master; file "/etc/namedb/blockeddomain.hosts";};</b></i><br>
<br>
With this loaded, any client request for <i><b>razdrochi.ru </b></i>will be redirected to whatever we have set up in <i><b>/etc/namedb/blockeddomain.hosts</b></i>. Essentially, all we are doing is mapping all of the domains listed in that file to the same DNS (A) record. <br>
<br>
Lets create this record now.<br>
<br>
6) Edit<b style="color:rgb(153,0,0)"> <i style="color:rgb(0,0,0)">/etc/namedb/blockeddomain.hosts</i></b>:<br>
<br>
<i><b>; This zone will redirect all requests back to the blackhole itself.<br>
<br>
$TTL&nbsp;&nbsp;&nbsp; 86400&nbsp;&nbsp; ; one day<br>
<br>
@&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; SOA&nbsp;&nbsp;&nbsp;&nbsp; </b></i><i><b style="font-family:arial,sans-serif">bhdns.mydomain.ca</b></i><i><b>. </b></i><i><b style="font-family:arial,sans-serif">bhdns.mydomain.ca</b></i><i><b>. (<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 1<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 28800&nbsp;&nbsp; ; refresh&nbsp; 8 hours<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 7200&nbsp;&nbsp;&nbsp; ; retry&nbsp;&nbsp;&nbsp; 2 hours<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 864000&nbsp; ; expire&nbsp; 10 days<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 86400 ) ; min ttl&nbsp; 1 day<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; NS&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; </b></i><i><b style="font-family:arial,sans-serif">bhdns.mydomain</b></i><i><b>.ca.<br>
<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; A&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 10.0.0.1<br>
<br>
*&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; A&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 10.0.0.1</b></i><br>
<br>
Save and exit the file.<br>
<br>
Note: You can redirect the request to anywhere you wish but it is worthwhile to send the user to a place that explains what just happened. If not, the user might get confused and open a vague "The Interweb is broken" helpdesk ticket.<br>
<br>
We should be good to go. Try and start the service: <br>
<br>
<i><b>~# /etc/rc.d/named start</b></i><br>
<br>
For debugging (or other) reasons it might be worth it to separate named logs from the syslog catchall. To do this, edit /etc/syslogd.conf and add:<br>
<br>
<i><b>!named<br>
*.*&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; /var/log/named.log</b></i><br>
<br>
Save and exit the file and then:<br>
<br>
<i><b>~# touch /var/log/named.log<br>
~# /etc/rc.d<span>/syslogd restart</span></b></i><br>
<br>
<br>
<br>
<font size="4"><b><a name="l3">How to automate the update process</a></b></font><br>
&nbsp;<br>
<br>
To make things a little easier, I made a simple script to manage the update process. The script does the following:<br>
<br>
1) Fetches the zonefile<br>
2) Performs a comparison with the current file, if there are no changes, exit. If there are then<br>
3) Make note of the additions/removals<br>
4) Put the new zonefile in place<br>
5) Restart the service<br>
6) Email the changes to an admin<br>
<br>
The script doesn't accommodate multiple files. If you will be using numerous sources just concatenate them into a single file. The script of course will need to be modified if you intend to run it on a system other than FreeBSD.<br>
<br>
You can grab the <a href="http://www.pintumbler.org/getzones.sh" target="_blank">script from here</a> or just follow the steps below.<br>
<br>
1) Download getzones.sh:<br>
<br>
<i><b>~# cd /etc/namedb/blackhole<br>
~# fetch http://www.pintumbler.org/getzones.sh<br>
</b></i>
<i><b>~# chmod +x /etc/namedb/blackhole/getzones.sh</b></i><br>
<br>
2) Create the temp directory:<br>
<br>
<i><b>~# mkdir /etc/namedb/blackhole/work</b></i><br>
<br>
3) Add an entry to roots crontab to run the script daily:<br>
<i><b><br>
~# crontab -e</b></i><br>
<br>
once the editor comes up, input the following line:<br>
<br>
<i><b>0&nbsp;&nbsp;&nbsp; 0 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; * &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; /etc/namedb/blackhole/getzones.sh &gt; /dev/null 2&gt;&amp;1</b></i><br>
<br>
Save and exit the file.<br>
<br>
This will update the file every day at midnight.<br>
<br>
My bash is pretty shoddy so you might want to test it out first :)<br>
<br>
<i><b>~# /etc/namedb/blackhole/getzones.sh</b></i><br>
<br>
If it doesn't work, send me an email and I will try and help you out.<font size="5"><b><br>
<br>
<font size="4"><a name="l4">How to setup Apache to provide an information page</a></font></b></font><br>
<br>
<br>
As I mentioned earlier, to avoid confusion it is a good idea to send the users to an information page. You can use any web server here, Apache is way overkill but its what I know. <br>
<br>
1) Install Apache. You can do this however you wish, I will just use the ports tree:<br>
<i><b><br>
~# cd /usr/ports/www/apache22; make install clean</b></i><br>
<br>
2) Edit <i><span style="color:rgb(0,0,0)"><b>/usr/local/etc/apache22/httpd.conf</b></span><span style="color:rgb(0,0,0)"> </span></i>and make the following changes (in order of appearance):<br>
<br>
<i><b>Listen 10.0.0.1:80<br>
ServerAdmin atech@mydomain.ca<br>
ServerName 10.0.0.1:80<br>
DocumentRoot "/usr/local/www/dnsbh"<br>
ErrorDocument 500 /404.html<br>
ErrorDocument 404 /404.html<br>
ErrorDocument 402 /404.html<br>
</b></i><br>
The rest of the defaults are fine.<br>
<br>
3) Create the web directory:<br>
<br>
<i><b>~# mkdir /usr/local/www/dnsbh<br>
<br>
</b></i>4) Create<i><b> <span style="color:rgb(0,0,0)">/usr/local/www/dnsbh/index.html</span></b></i><i><b> </b></i>this will be the main landing page when folks are redirected:<br>
<br>
<b><i><span>&lt;!DOCTYPE html&gt;</span><br>
&lt;html&gt;<br>
&lt;head&gt;<br>
&lt;title&gt;Your Org Name - IT services&lt;/title&gt;<br>
&lt;script type="text/javascript"&gt;<span style="font-family:arial,sans-serif">url = parent.window.location.href;</span><span style="font-family:arial,sans-serif">&lt;/script&gt;</span><br>
<span style="font-family:arial,sans-serif">&lt;/head&gt;</span><br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">&lt;body&gt;</span></i>
<i><br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">&lt;h3&gt;Security Notice...&lt;/h3&gt;</span></i>
<i><br style="font-family:arial,sans-serif">
<br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">You have been redirected to this page because the website that you tried to visit has been known to harbor</span></i>
<i><br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">or distribute Spyware, Viruses or other forms of malicious software.</span></i>
<i><br style="font-family:arial,sans-serif">
<br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">&lt;br&gt;&lt;br&gt;</span></i>
<i><br style="font-family:arial,sans-serif">
<br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">As part of our Information Security Policy we maintain a listing of potentially harmful sites to assist in the protection and stability of our computing resources. This is also done to protect users from divulging personal information to third parties where it could be used for illicit purposes such as Spam or Fraud.</span></i>
<i><br style="font-family:arial,sans-serif">
<br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">&lt;br&gt;&lt;br&gt;</span></i>
<i><br style="font-family:arial,sans-serif">
<br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">If you feel your access to this web site is a requirement, contact your local Information Technology Services department for assistance.</span></i>
<i><br style="font-family:arial,sans-serif">
<br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">&lt;/body&gt;</span></i>
<i><br style="font-family:arial,sans-serif">
<span style="font-family:arial,sans-serif">&lt;/html&gt;</span></i>
</b><br>
<br>
<div>
<div>Save and exit the file.<br>
<br>
5) Create <i><b>/usr/local/www/dnsbh/warn.png</b></i>. This image should be around 127 X 57 pixels and contain something that identifies your organization along with something that conveys 'warning' either through words or images.<br>
<br>
What happens is a lot of ad space can point to blocked domains and while the core site loads OK, you end up with your warning index document trying to pack into a small ad space. There might be a better way around this but the simplest solution I found is to use a 404 document which displays the image and that is linked back to the main page. <br>
<br>
It is enough to prod the user along.<br>
<br>
6) Create <i><b>/usr/local/www/dnsbh/404.html</b></i>. This should look something like:<br>
<br>
<i><b><span>&lt;!DOCTYPE html&gt;</span><br>
&lt;html&gt;<br>
&lt;head&gt;<br>
&lt;body&gt;<br>
&lt;a href="/index.html" target="_new"&gt;&lt;img border="0" src="/warn.png"&gt;&lt;/a&gt;<br>
&lt;/body&gt;<br>
&lt;/html&gt;<br>
</b></i><br>
<br>
That's it. There is obviously a lot of options here, this should be enough to get you started.<br>
<br>
<br>
<br>
<font size="4"><b><a name="l5">Monitoring examples and ideas</a></b></font><br>
<br>
I have some stuff set up that leverages our existing Netflow data to try and add a little value to what goes on with this box.<br>
The examples below are simply trending connections to port 80. <br>
<br>
The shortcomings of this solution aside, that second image is quite compelling. You don't even need an analyst to interpret it. This is the kind of stuff that can be easily offloaded because the message is so poignant.<br>
<br>
This example shows a typical day of activity:<br>
<br>
<div style="display:block;text-align:center"><a href="http://www.pintumbler.org/Code/dnsbl/dnsbh1.png?attredirects=0" imageanchor="1"><img border="0" src="http://www.pintumbler.org/_/rsrc/1303426214049/Code/dnsbl/dnsbh1.png"></a></div>
<br>
<br>
This shows an infection:<br>
<br>
<br>
<div style="display:block;text-align:center"><a href="http://www.pintumbler.org/Code/dnsbl/dnsbh2.png?attredirects=0" imageanchor="1"><img border="0" src="http://www.pintumbler.org/_/rsrc/1303426235351/Code/dnsbl/dnsbh2.png"></a></div>
<br>
<br>
A nice summary table like the one below is very useful. It is important to not just focus on hits but just how much data a client trying to send out. Large payloads repeatedly sent to this device should be an instant alarm.<br>
<br>
<br>
<div style="display:block;text-align:center"><a href="http://www.pintumbler.org/Code/dnsbl/dnsbh3.png?attredirects=0" imageanchor="1"><img border="0" src="http://www.pintumbler.org/_/rsrc/1303426262027/Code/dnsbl/dnsbh3.png"></a></div>
<br>
Keep in mind too that that this is a webserver people are connecting to. Which means: LOGS!&nbsp; Aside from the fact that we can use these to clarify events, the data is screaming to be mined. I am not quite there yet, but soon.<br>
<br>
<font size="4"><b><br>
<a name="l6">Summary</a></b></font><br>
<br>
Being a little short handed at work, this was one of the first security solutions that I gravitated towards. The passiveness (and price) just made sense. A DNSBH can dramatically improve an organizations overall security posture for next to nothing. Yes, understand that It is NOT going to help you deal with intelligent threats, but you know what? that doesn't really matter because most aren't. <br>
<br>
The sludge, the unworthy.. that is where this solution shines. If you are short on time, and short on resources, this is a gift.<br>
<br>
Thanks to everyone that has contributed to and maintained these lists over the years. You have made my life a lot easier :)<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
</div>
</div></div></td></tr></tbody></table>
</div>
