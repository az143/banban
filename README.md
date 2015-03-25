# banban, a variation of the fail2ban theme

in this repository you'll find banban, a small tool that provides a
variant and subset of [fail2ban](http://www.fail2ban.org)
functionality that i personally consider useful, massaged and extende 
with stuff that f2b does _not_ do. banban relies directly and solely on
iptables' recent match, and reacts close to realtime.

banban is also ridiculously lean compared to f2b: 200 lines of perl,
70 lines of config.

## how it works

banban listens to syslog information on a UNIX domain socket. whenever
it finds something that belongs to 'goodpatterns' it unblocks the
source IP address. whenever it finds something in a variety of 'badpatterns',
sufficiently often, it blocks the source IP address.

banban works fine with both ipv4 and ipv6 addresses.

blocking is done via writing the IP address to /proc/net/xt\_recent/_name_,
ie. telling the iptables recent match to consider this a good/bad
thing. you will therefore need a few iptables rules that enforce the
blocks that banban simply "primes".

a set of 'bad stuff' rules can specify how many occurrences of 
rules from the set have to occur within an interval of X seconds
for banban to fire.

banban can also use conntrack to immediately reset and dump existing
connections with offending parties.

that's about all there is to it.

## configuration and integration

the example banban.example.yml is pretty much self-explanatory.

to get syslog-ng 3.3.X to send data to a UNIX domain socket, 
i use this snippet:

	destination banban { unix-dgram("/var/lib/syslog-ng/banban" 
	flags(no_multi_line) flush_lines(1) 
	template("$DATE $HOST $MSGHDR$MESSAGE\n")); };
	
one caveat: with syslog-ng the listener, banban, should be started _first_
as syslog-ng doesn't create the socket - but at least it retries and 
reconnects, if not exactly quickly or without complaining a lot.

to get rsyslog to send data to a UNIX domain socket,
i use this bit of config:

	# use unix domain socket for faster comms with banban
	$ModLoad omuxsock
	$OMUxSockSocket /var/spool/rsyslog/banban
	$OMUxSockDefaultTemplate RSYSLOG_TraditionalFileFormat

	*.* :omuxsock:
	
this is tested and works with rsyslog 5.8.X.

to get the kernel to enforce your blocks, you'll need firewall rules
somewhat similar to these examples:

        # brief throttle
        iptables -A blacklist -m recent --name THROTTLE --rcheck --hitcount 1 \
            --seconds 180 -j DROP
        # full blocks, smtp or otherwise
        BOTH -A blacklist -m recent --name BLOCKER --rcheck --hitcount 1 \
            --seconds 172800 -j DROP
		# plus the same with ip6tables...

these are relative to a custom chain called blacklist, which is jumped to 
from INPUT under various circumstances. the recent names BLOCKER and THROTTLE
correspond to the "target"s in banban's config.

note that banban adds an ip to target X on blocking, but removes an ip address
from **all** known target Xs on unblocking.

## copyright, license

all this stuff is (c) 2013-2015 Alexander Zangerl <az@snafu.priv.at>
and licensed under the Gnu General Public License version 2.

share and enjoy `:-)`


