# bgpfeeder
by Matthew Bloch <matthew@bytemark.co.uk>

bgpfeeder is a program to feed a BGPv4 speaker with an infrequently
changing list of static internal routes.  We use it at Bytemark to
rearrange routing information from our "network organiser" database,
to avoid staff having to log in to core routers to effect common
routing changes.  We also use it to communicate with transit providers
to communicate "black hole" information about traffic we don't want
passed on to our routers.

## Quick start

bgpfeeder is portable and should run on any system that has Ruby 1.8
installed (though only tested on Linux).  Here's an example of how to
use it:

    #
    # These are the routes I'd like distributed:
    #
    echo '
    192.168.0.0/24 via 10.0.0.1
    192.168.1.0/24 via 10.0.0.2
    192.168.2.0/24 via 10.0.0.3 communities 65534:10 local_pref 50
    ' > routes

    #
    # Here are the BGP peers that I'd like them distributed to:
    #
    echo '80.68.80.2' > peers

    #
    # Start bgpfeeder as AS 65534, BGP ID 80.68.80.1 and set the local_pref
    # value to 200 unless otherwise specified.
    #
    ./bgpfeeder --as-number 65534 --bgp-identifier 80.68.80.1 --local-pref 200

    #
    # Go ahead and change the routes / peers files after the program has
    # started, it will pick up any changes.


## How to use

bgpfeeder is configured with a set of static parameters on the command
line which cannot be changed once started.  These are your routers' AS
number and BGP identifier (which is usually the IPv4 address of the
router).  You can also set the local_pref value for the routes and the
list of communities to add to every route which may be useful extra
information for your routers to filter on.  Finally there is the hold
time which defaults to 60s, and some logging settings.

The administrator also specifies filenames of two database files: the peers
file, and the routes file.  These are the two lists which can change while
the program is running.

Here is the help:

    Usage: ./bgpfeeder [options...]

    Options (must specify as-number and bgp-identifier at least):
        -h, --help                       Show this message
        -a, --as-number                  Set local AS number (default 65534)
        -i, --bgp-identifier IP.ADDRESS  Set BGP identifier
        -r, --routes FILE                Set routes file (default routes)
        -p, --peers FILE                 Set peers file (default peers)
            --local-pref PREF            Set local preference (default 100)
            --hold-time SECS             Set hold time (default 60)
            --communities a,b,c          Set list of communities (default _)
            --log-level LEVEL            Set log level (default INFO)
            --log-file FILE              Set log file (default STDOUT)

The peers file contains a list of BGP peers (and TCP-MD5 passwords for each one).

When a new peer is seen for the first time, the program continually
tries to connect to it and send its current complete route list.  When
a peer is removed from the list, the BGP connection is terminated.
Peers are IPv4 or IPv6 addresses, and can have a port specifier
i.e. all of these are valid:

* 80.68.80.1
* 127.0.0.1:1234
* fc00:1234::1
* [fc00:1234::1]:1234

The port syntax can be useful for testing (though no real BGP speaker
will listen on a port other than the default 179).  You can also
specify an optional second parameter after a space, which is the
TCP-MD5 password e.g.  _80.68.80.1 my_session_secret_ would specify
peer 80.68.80.1 with the password my_session_secret.  If no password
is specified, an ordinary TCP socket will be used.

You can also specify a source address (and port) if you want,
e.g. here is the above list with an example source address supplied
for each.

* 80.68.80.128-80.68.80.1
* 127.0.0.1:9999-127.0.0.1:1234
* fc00:1234::400-fc00:1234::1
* [fc00:1234::400]:9999-[fc00:1234::1]:1234

The routes file contains a list of static routes with "next hop"
specifiers.  These can either be in roughly "Linux" or "Cisco"
formats; e.g. any of these means the same thing:

    10.0.0.0/24 192.168.0.1
    10.0.0.0/24 via 192.168.0.1
    10.0.0.0 255.255.255.0 192.168.0.1
    ip route 10.0.0.0 255.255.255.0 192.168.0.1

i.e. routing the IP block _10.0.0.0/24_ via the next hop of
_192.168.0.1_. 

Duplicate routes in the route file will be ignored.  You may also
specify IPv6 routes in the usual formats:

    aaaa:bbbb:cccc:dddd::/64 fc00:1234::1
    aaaa:bbbb:cccc:dddd::/64 via fc00:1234::1

As well as the obvious NEXT_HOP attribute (3), the BGP UPDATE messages sent
by bgpfeeder contain an ORIGIN setting indicating that these have been
learned by an IGP.

For "exterior" BGP, where the peer's AS# is different from our local one,
we send our AS as the full path, and no local preference.

For "interior" BGP, where we're in the same AS as our peer, we send a zero-
length AS path, and a local preference attribute.

You can specify local_pref and communities values on a route-by-route basis, 
e.g.

    10.0.0.0/24 via 192.168.0.1 communities 65534:111
    10.0.1.0/24 via 192.168.0.2 communities 65534:111,0xff001122 local_pref 50

The local_pref in the routes file will always override the value set
on the command line.  However communities set on the command line will
always be present, and those specified in the routes file will be
appended to the list.

The routes file is checked for differences whenever it is changed and
UPDATE messages sent to current peers accordingly.  When routes are
removed, the message puts the route into the 'withdrawn' section.
When routes are replaced, the same UPDATE message contains both the
withdrawn route and the new new one so the peer can update its
database without a "flap" (i.e.  removing the route and adding it
again).

IPv6 route updates are always sent using the multiprotocol extensions,
and IPv4 updates are always sent using "plain" BGPv4 UPDATE messages,
even if the peer supports multiprotocol extensions (I'm not sure which
is customary, this is just what wseemed to work against an IPv4 and
IPv6 neighbour specification under IOS).

The peers file is checked for differences whenever it is changed; new peers
are started within a second.  Old peers are signalled to close straight away
but may take a few seconds to close down properly.  Exactly-specified
duplicates will be ignored.



## Multiple sets of routes & peers, IPv6 support

When you have established a BGP session to a peer over IPv6, you
probably want to avoid sending IPv4 routes over the same connection.
To avoid having to start lots of copies of bgpfeeder, you can instead
specify --routes and --peers multiple times, and maintain separate
databases of IPv4 and IPv6 routes, e.g:

    ./bgpfeeder ... --routes r4 --peers p4 \
                    --routes r6 --peers p6

tells the program to send the routes in the file 'r4' to the peers in
file 'p4' and the same for files 'r6' and 'p6'.

There is nothing stopping you from using as many -r and -p directives
as you need if you wanted to maintain lots of separate bgpfeeder
sessions.

If you try to feed IPv6 routes to a peer that does not understand BGP
multiprotocol extensions, those routes will be silently ignored.

When bgpfeeder first connects to any peer, it declares its own
multiprotocol support.  If gets a Notification back that suggests the
peer doesn't understand IPv6, it will disconnect and try again
without.


## Limitations

There should be no limits on the size of the routes and peers files
other than what the system imposes.  The routes file is re-read and
parsed in its entirety when it is changed, so updates may be slower to
get into the BGP network the larger the file is.  However this is by
design; the program is not intended to serve very frequenty changing
routes (i.e. more than about once per minute).

bgpfeeder is not a "proper" BGPv4 daemon in that it doesn't listen on
port 179 for incoming connections, and discards any UPDATE messages
from its peers.  Its state machine bears no relation to the
complicated one described in RFC 1771.  None of this should affect it
doing its job.

Received UPDATE messages generate a warning as the system is written
in a scripting language which is relatively slow to handle large
amounts of data.  If a peer is bombarding it with an entire internet
routing table when it starts up, it will take a lot of CPU time to
discard such information relative to a normal BGP program.

The program is pretty dim and basically single threaded (despite using
Ruby-internal threads in a couple of places) so if it has to process
large amounts of data on a particularly busy system, it may risk
tripping hold times more than other BGP daemons might.  The logs
should make clear when this happens but I'm not sure how big a risk
this is.

## Known bugs

 * "Expected OPEN, got  instead" log message - peers don't open first time, 
   should work again after retry 20s later.
 
 * It will eat memory in proportion to the number of changes in the routes
   file because it keeps a complete update log in RAM.

 * TODO: STDIN mode, allow user to make their own route and peer  updates
   fed from an external tool?

 * Probably won't always detect when the BGP stream is corrupt, and will
   crash out with a NoMethodError or something ugly.

## Deployment example

If you were running the program on the IP 80.68.80.1, you might use a script
like this:

    cd /home/bgpfeeder
    ./bgpfeeder --as-number 65534 --bgp-identifier 80.68.80.1 \
      --host-time 60 --local-pref 100  --routes routes --peers peers

The program does not daemonise itself - an example init script is provided
as bgpfeeder.init.d or you can make your own arrangements.

You can set the environment variable BGPFEEDER_LOG_LEVEL to one of DEBUG,
INFO, WARN or ERROR to make the logging more or less verbose.  The default
is WARN.

It is important that the routes and peers files are updated _atomically_,
i.e.

 * DO use rsync if you are copying the peers & routes file on remotely
 * DO write to (for instance) peers.tmp and then "mv peers.tmp peers"
 * DON'T edit the peers or routes file directory with vi, or scribble to
 them with "echo 1.2.3.4 >>peers"

If you don't follow this advice you may find routes or peers flapping
unpredictably, as bgpfeeder won't care if your file is only half written.

If you're talking to a core BGP router with a full routing table you may
want to make sure that it never sends prefixes back to bgpfeeder with a
configuration snippet like this (either quagga or Cisco):

      access-list 100 deny ip any any
      router bgp xxx
         neighbor yyy distribute-list 100 out

Where xxx is your AS number and yyy is the BGP ID of your bgpfeeder
instance.  

## Testing against quagga

A half-finished test rig is included for use on Debian systems.  If you
"rake test:qemu_shell" the system will build a Debian test system with a
suitably configured BGP daemon, then dump you in a shell.  

In the qemu window, run:

    /usr/lib/quagga/bgpd &
    vtysh
    debug bgp updates
    debug bgp events

If you're not using qemu and setting up your own bgpd, I only needed the
following file in `/etc/quagga/bgpd.conf`:

    log stdout debugging
    router bgp 65534
      neighbor 10.0.2.2 remote-as 65534

Then in another window, run:

    touch routes peers
    BGPFEEDER_LOG_LEVEL=DEBUG ./bgpfeeder 65534 10.0.2.2 60 100 routes peers

You should now be able to see the systems start up and warn that they are
doing nothing.  Verify that "show ip bgp" shows no BGP peers.  Now add
_10.0.2.15:1179_ to the peers file and watch it connect.  Now try adding
and
deleting routes to the routes files, or add some duff peers etc. to see the
system work.


## Testing against Cisco IOS

I have only tried with dynamips and a Cisco 7200 image, which I started with
this command line:

      ./dynamips image.bin -t npe-400 \
        -p 1:PA-A1 -p 2:PA-8T -p 3:PA-4E -p 4:PA-POS-OC3 -p 6:PA-FE-TX \
        -s 0:0:tap:c7200
      ip link set c7200 up
      ip addr add 10.0.0.2/24 dev c7200
      ip addr add fc00:1234::2/64 dev c7200

Then on the IOS command line (assuming a completely fresh router
configuration):

     enable
     configure terminal
      ip routing
      ipv6 unicast-routing
      ipv6 cef
    
     interface FastEthernet0/0
      ip address 10.0.0.1 255.255.255.0
      ipv6 address fc00:1234::1/64
      no shutdown
      exit
    
     router bgp 65534
      neighbor 10.0.0.2 remote-as 65534
      neighbor 10.0.0.2 password TopSecret
      neighbor fc00:1234::2 remote-as 65534
      neighbor fc00:1234::2 activate
      exit
    exit
    write memory

You should now be able to ping 10.0.0.1 to talk to your virtual Cisco.  

Then run bgpfeeder as before but your peer address will be simply
"10.0.0.1 TopSecret"; the IOS output is very helpful when things go wrong, 
and you can "show ip bgp" to see which routes are being received.  I also 
find that "debug ip bgp all", "debug ip bgp events" and "debug ip bgp updates" 
are very useful to have turned on.

# Copyright

bgpfeeder is (C) Bytemark Hosting 2009-2016, and may be distributed according 
to the terms of the GNU General Public License version 3 ONLY.


