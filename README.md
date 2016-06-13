# High Availability with OSPF Backed Anycasting

'load balancers' are typically the bane of all system administrators as when they are deployed more often for their advertised failover capabilities you quickly find that the medicine is worse than the disease.  Load balancers rarely are very RFC-caring and of course ironically create single points of failure.  Their functioning results in a lot more complexity being added to whatever system they are meant to be making highly available which of course leads to fails modes that are far more severe when things do crash and burn.  In short, load balancers are a prime example of what is wrong with sysadmin'ing mindset now-a-days, instead of building a system that gracefully fails and expects componment failure, all resources are spent on making sure nodes do not fail. 

[Anycast'ing](http://en.wikipedia.org/wiki/Anycast) is a great method to add failover to a system that uses an [IGP](http://en.wikipedia.org/wiki/Interior_gateway_protocol) such as [OSPF](http://en.wikipedia.org/wiki/Open_Shortest_Path_First) to get client traffic to the nearest service node.  IGP's are widely understood and easy to diagnose (compared to a vendor turn-key 'blackbox'), if you already run one at your organisation then this could be a perfect and cheap solution for you.

As with all High Availability (HA) systems, everything boils down to a good and simple probe that tests if the service on the local node is functioning correctly.  Often surprising is that you will find 90% of your time will be spent producing nothing more than a sixty line shell script that performs just this job.

The job of the, in the below DNS resolver example, shell script is to test if the local resolver is functioning.  If so then the 'service' IP address is added to the node which prompts the IGP daemon to begin advertising the address to the network.  If the service is down, then the service IP is removed and the IGP daemon withdraws the advertisment and traffic to that service IP is rerouted to the next nearest functioning node.  This is great for unreliable services as you can very simply get very reliable systems by creating duplicate service nodes.  For example, say your DNS server only functions 80% of the time (down for 20%), by having four DNS servers you get a service that gives you nearly three nines uptime (''1-(0.20^4) = 0.9984'').

**N.B.** it is *strongly* recommended you have at least three L3 IP routing hops (ie. traceroute has more than three entries) between your anycast'ed nodes so that you avoid your OSPF topology settling on an equal-cost-multipath routing table.  This means that a router on your network sees that the path to your anycast'ed address is the same across different links and will load balance on a per-packet basis (//very// bad for TCP flows, DNS *is* TCP also remember especially with DNSSEC).  If you cannot arrange your network to have at least three hops, then increase the routing metric on the directly connected link to one of your anycast'ed nodes to act as a work around.

## OSPF

Below we give an example of configuring a single service node ([quagga](http://www.quagga.net/) capable host) to provide a DNS resolver for clients of the network.  The node communicates with it's upstream Cisco router (the example shown works on a C6500 and also a C3750 running 'IP base'); do send me configuration snippets for other routers.  The subnet the service node and router lives on 198.51.100.0/30 and 2001:db8:ffff:1000::/64 whilst the service IP's are 192.0.2.1 and 2001:db8:ffff:a000:fdc0::3b5.

### Quagga

Tinker with '/etc/quagga/daemons' and then:

    # cat `<<EOF >` /etc/quagga/zebra.conf
    password foobar
    EOF
    
    # chmod 640 /etc/quagga/zebra.conf
    # chown quagga:quagga /etc/quagga/zebra.conf


    
    # cat `<<EOF >` /etc/quagga/ospfd.conf
    password foobar
    !
    interface bond0
     ip ospf hello-interval 5
     ip ospf dead-interval 20
    !
    router ospf
     ospf router-id 198.51.100.2
     log-adjacency-changes
     redistribute connected
     network 192.0.2.1/32 area 0.0.0.1
     network 198.51.100.0/30 area 0.0.0.0
     distribute-list 50 out connected
    !
    access-list 50 permit 192.0.2.1
    !
    line vty
    EOF
    
    # chmod 640 /etc/quagga/ospfd.conf
    # chown quagga:quagga /etc/quagga/ospfd.conf

    # cat `<<EOF >` /etc/quagga/ospf6d.conf
    password foobar
    !
    debug ospf6 lsa unknown
    !
    interface bond0
     ipv6 ospf6 cost 1
     ipv6 ospf6 hello-interval 5
     ipv6 ospf6 dead-interval 20
     ipv6 ospf6 retransmit-interval 5
     ipv6 ospf6 priority 1
     ipv6 ospf6 transmit-delay 1
     ipv6 ospf6 instance-id 0
    !
    router ospf6
     router-id 198.51.100.2
     redistribute connected route-map filter
     area 0.0.0.0 range 2001:db8:ffff:1000::/64
     area 0.0.0.1 range 2001:db8:ffff:a000::/64
     interface bond0 area 0.0.0.0
    !
    ipv6 prefix-list dns seq 5 permit 2001:db8:ffff:a000:fdc0::3b5/128
    !
    route-map filter permit 10
     match ipv6 address prefix-list dns
    !
    route-map filter deny 20
    !
    line vty
    EOF
    
    # chmod 640 /etc/quagga/ospf6d.conf
    # chown quagga:quagga /etc/quagga/ospf6d.conf

### Cisco
    
    ip access-list standard ospfv4-dns
     permit 192.0.2.1
    !
    router ospf 30000
     router-id 198.51.100.1
     log-adjacency-changes
     passive-interface default
     no passive-interface Port-channel1
     network 192.0.2.1 0.0.0.1 area 1
     network 198.51.100.0 0.0.0.3 area 0
     distribute-list ospfv4-dns in
    !
    !
    ipv6 prefix-list ospfv6-dns seq 5 permit 2001:DB8:FFFF:A000:FDC0::3B5/128
    !
    ipv6 router ospf 30000
     router-id 198.51.100.1
     log-adjacency-changes
     area 0 range 2001:DB8:FFFF:1000::/64
     area 1 range 2001:DB8:FFFF:A000::/64
     distribute-list prefix-list ospfv6-dns in  
     passive-interface default
     no passive-interface Port-channel1
    !
    !
    interface Port-channel1
     description dns-server
     no switchport
     ip address 198.51.100.1 255.255.255.252
     ip ospf hello-interval 5
     ipv6 address FE80:: link-local
     ipv6 address EXAMPLE 2001:db8:ffff:1000::/64 anycast
     ipv6 nd router-preference High
     ipv6 ospf hello-interval 5
     ipv6 ospf 30000 area 0
    end

It is *recommended* that you use your global IGP instance to do this all with, but for those cursed with legacy EIGRP deployments (or using iBGP/ISIS and want to decouple voliate anycast'ed nodes) you will need to run a separate routing process and then 'redistribute' these new OSPF processes into your global process, for example:

    router eigrp 1234
     redistribute ospf 30000 metric 1 1000 255 1 1500
    
    ipv6 router ospf 1234
     redistribute ospf 30000

## Probes

As mentioned earlier, the tricky part is the probe script, I have made available the ones I developed and use at for [our organisation](http://www.soas.ac.uk/):

 * [`dns-probe`](dns-probe) - recursive servers only
 * [`radius-probe`](radius-probe)
 * [`syslog-probe`](syslog-probe) - [syslog-ng](http://www.balabit.com/network-security/syslog-ng) with [netcat6](http://www.deepspace6.net/projects/netcat6.html)
 * [`squid-probe`](squid-probe)
 * [`tcp-route-probe`](tcp-route-probe)

It is simply a case of dropping the script into '/usr/local/sbin/' and running it as root.  To automatically start the script at boot time, add something like the following line to '/etc/rc.local':

    nohup /usr/local/sbin/dns-probe || true

The scripts support signal handling too:

 * **`SIGTERM`:** remove HA IP address(es) and exit cleanly
 * **`SIGTSTP`:** remove HA IP address(es) and send ''SIGSTOP'' to self
 * **TODO:** deadvertise the service by [persuading Quagga to increase the routing metric first to 65535](http://routerjockey.com/2011/02/21/ospf-graceful-shutdown/) and then after a period of time to remove the service IP.  On second thought this is more than likely not approiate as your node is an endpoint rather than a conduit so the effect would be the same as simply tearing down the IP
* **''SIGCONT'':** informs the kernel to resume the process and the probes will start afresh

### DNS (unbound)

This probe/IGP approach is very helpful when you need to perform maintainance on your nodes, such as needs to be done in my [Protecting Users with DNS Malware Blacklisting](http://www.digriz.org.uk/dns-malware-blacklisting) by the regular cron job described there.  An example of this is when 'unbound' restarts it initially takes some time to get going, especially so on some low end [hardware](http://www.digriz.org.uk/kirkwood), whilst pre-loading the 16000 domains that need hijacking.  To hide this delay is trivial to mask with anycast'ing:
 1. 'suspend' the probe and remove the service IP's (''pkill -TSTP dns-probe'')
 1. perform the maintainence work (''/etc/init.d/unbound restart'')
 1. resume the probe and have it add the service IP's (''pkill -CONT dns-probe'')

In this case, the cron job (''/etc/cron.d/local-unbound'') is amended as follows:
    
    #MAILTO=hostmaster
    PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
    
    # pre-ha enabled
    #17 0-23/6  * * * root   nice -n10 blacklist2dns && nice -n5 unbound-checkconf && /etc/init.d/unbound restart
    
    # ha enabled
    17 0-23/6   * * * root   nice -n10 blacklist2dns && nice -n5 unbound-checkconf && pkill -TSTP dns-probe && /etc/init.d/unbound restart && pkill -CONT dns-probe
    
    # ha enabled utilising http://habilis.net/cronic/
    # 17 0-23/6 * * * root    cronic sh -c 'nice -n10 blacklist2dns; RC=$?; [ $RC -eq 1 ] && exit 0; [ $RC -gt 1 ] && exit $RC; nice -n5 unbound-checkconf && pkill -TSTP dns-probe && /etc/init.d/unbound restart && pkill -CONT dns-probe'

### RADIUS (freeradius)

**N.B.** work in progress

### /etc/freeradius/sites-available/viewpoint
    
    server viewpoint {
            listen {
                    type            = status
                    port            = 18120
                    ipaddr          = 127.0.0.1
                    interface       = lo
            }
    
            client 127.0.0.1 {
                    shortname       = localhost
                    secret          = foobar
            }
    
            authorize {
                    ok
    
                    # respond to the Status-Server request.
                    Autz-Type Status-Server {
                            ok
                    }
            }
    }

### syslog (syslog-ng/netcat6)

**N.B.** work in progress

### /etc/syslog-ng/probe.conf
    
    source s_probe { udp(ip(127.0.0.1) port(5514)); };
    destination d_probe { udp("127.0.0.1" port(5515) flush_lines(0)); };
    log { source(s_probe); destination(d_probe); };

### HTTP Proxy (squid)

**N.B.** work in progress
    
    ip access-list standard ospfv4-proxy
     permit 193.63.73.35
     permit 193.63.73.34
     permit 193.63.73.33
    
    ip access-list standard ospfv4-proxy-remote
     permit 193.63.73.34
    !
    route-map ospfv4-proxy permit 10
     match ip address ospfv4-proxy-remote
     set metric 10000 4000 255 1 1500
    route-map ospfv4-proxy permit 20
     set metric 20000 2000 255 1 1500
    
    router ospf 30002
     router-id 212.219.238.13
     passive-interface default
     no passive-interface Port-channel7
     network 193.63.73.33 0.0.0.0 area 0.0.0.1
     network 193.63.73.34 0.0.0.0 area 0.0.0.1
     network 193.63.73.35 0.0.0.0 area 0.0.0.1
     network 212.219.238.12 0.0.0.3 area 0
     distribute-list ospfv4-proxy in
    
    
    ipv6 prefix-list ospfv6-proxy seq 10 permit 2001:630:1B:1001:FCE9:4633:2F26:4A5B/128
    ipv6 prefix-list ospfv6-proxy seq 20 permit 2001:630:1B:1001:3B30:4100:D401:2A97/128
    ipv6 prefix-list ospfv6-proxy seq 30 permit 2001:630:1B:1001:A0D3:5619:9943:8390/128
    
    ipv6 prefix-list ospfv6-proxy-remote seq 10 permit 2001:630:1B:1001:3B30:4100:D401:2A97/128
    !
    route-map ospfv6-proxy permit 10
     match ipv6 address prefix-list ospfv6-proxy-remote
     set metric 200
    route-map ospfv6-proxy permit 20
     set metric 10
    
    ipv6 router ospf 30002
     router-id 212.219.238.13
     area 0 range 2001:630:1B:6007::/64
     area 0.0.0.1 range 2001:630:1B:1001:3B30:4100:D401:2A97/128
     area 0.0.0.1 range 2001:630:1B:1001:A0D3:5619:9943:8390/128
     area 0.0.0.1 range 2001:630:1B:1001:FCE9:4633:2F26:4A5B/128
     distribute-list prefix-list ospfv6-proxy in
     passive-interface default
     no passive-interface Port-channel7
    
    router eigrp 111
     redistribute ospf 30002 route-map ospfv4-proxy
    
    ipv6 router ospf 10
     redistribute ospf 30002 route-map ospfv6-proxy

    chinua# show run
    
    Current configuration:
    !
    password foobar
    !
    interface bond0
     ip ospf hello-interval 5
     ip ospf dead-interval 20
    !
    interface eth0
    !
    interface eth1
    !
    interface lo
    !
    router ospf
     ospf router-id 212.219.238.14
     log-adjacency-changes
     redistribute connected
     passive-interface default
     no passive-interface bond0
     network 193.63.73.33/32 area 0.0.0.1
     network 193.63.73.34/32 area 0.0.0.1
     network 193.63.73.35/32 area 0.0.0.1
     network 212.219.238.12/30 area 0.0.0.0
     distribute-list 50 out connected
     default-information originate
    !
    access-list 50 permit 193.63.73.33
    access-list 50 permit 193.63.73.34
    access-list 50 permit 193.63.73.35
    !
    line vty
    !
    end

### TCP Route Prober (netcat6)

A slightly different problem to be solved here.  [SOAS](http://www.soas.ac.uk/) has a handful of IPsec tunnels and to monitor the services at the far end of the tunnel could only realistically be done (for braindead reasons and constraints placed on us by the venduh) with netcat in 'scanning mode' ('-z').

Another amendment is that we are advertising now kernel installed routes and not connected IP's and/or subnets:

    router ospf
     ospf router-id 198.51.100.2
     log-adjacency-changes
     redistribute kernel
     network 192.0.2.153/32 area 0.0.0.1
     network 192.0.2.201/32 area 0.0.0.1
     network 192.0.2.211/32 area 0.0.0.1
     [snipped]
     network 198.51.100.0/30 area 0.0.0.0
     distribute-list 50 out kernel

The probe initialising looks like:

    ## northgate probes
    # greenford
    nohup /usr/local/sbin/tcp-route-probe bond0 192.0.2.153 80 443 || true
    #nohup /usr/local/sbin/tcp-route-probe bond0 192.0.2.201 22 23 992 || true
    #nohup /usr/local/sbin/tcp-route-probe bond0 192.0.2.211 1521 1526 || true
    # sqp
    nohup /usr/local/sbin/tcp-route-probe bond0 198.51.100.153 80 443 || true
    #nohup /usr/local/sbin/tcp-route-probe bond0 198.51.100.201 22 23 992 || true
    #nohup /usr/local/sbin/tcp-route-probe bond0 198.51.100.211 1521 1526 || true

If *all* the TCP ports are open, a three way TCP handshake can take place, then the script will add a route to that IP in the kernel via the interface specified on the command line ('bond0' in the above examples).  Traffic flow could be considered a little weird as client traffic comes in over 'bond0', then is NAT'ed, IPsec'd and sent back out over 'bond0'.  This works as the far end tunnel endpoint is not routed to the onsite local node.  No doubt a unique problem experienced by few people, but the script it's-self no doubt will be useful to others wishing to roll their own probe.
