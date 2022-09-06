---
title: "I can't reach a published container port from outside of the host!"
date: 2022-09-06T22:34:32+02:00
tags: ["iptables", "docker", "troubleshooting"]
summary: >
    After a seemingly fresh install of docker I couldn't access an exposed container
    port. This post documents the troubleshooting process and the culprit.
---

I have installed Ubuntu Server 22.04 on one of my machines and tried to launch a
simple docker-compose stack.

`docker-compose` came installed together with the `docker` snap, which resulted
in some issues (same as [here](
https://askubuntu.com/questions/1374480/docker-compose-installed-with-snap-gives-error-on-yml-file)),
so I quickly decided to simply replace that with the `docker.io` APT package:
```
# snap remove docker
docker removed
# apt-get install -y docker.io docker-compose
...
# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
#
```
Done.

Now back to running my precious containers...
`docker-compose up`, I'm going to the right url in my browser and... nothing.

Ok, this can't be hard to fix, right? Right?!

Follow along, let's see how soon you'll figure out what the issue is!

## The setup

Firstly, let's make it clear what the setup is. The Ubuntu box is a vanilla
installation, the only additional steps were the removal of the `docker` snap
and the installation of the `docker.io` package. I'm running a container with an
exposed (published) port and trying to access it from my workstation.

The Ubuntu box (let's call it `dockerhost`) and my workstation (`mypc`) are in
different subnets, with a router in between that should allow the traffic to
pass between them.

```goat

+------+  192.168.30.0/24  +--------+ 192.168.50.0/24  +------------+
| mypc |-------------------| Router |------------------| dockerhost |
+------+                   +--------+                  +------------+
```

Unfortunately for some reason the connection simply times out...

## Is it the container?

I haven't used the container from the docker-compose definition before, so I'm
replacing it with something trivial to see if that helps:
```
root@dockerhost:~# docker run -d --rm -p 8080:80 nginx
948a0b20cc19bf0730503b9f520d999ad2a2c2faf8201dd6326acd6469655eb0
root@dockerhost:~# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                                   NAMES
948a0b20cc19   nginx     "/docker-entrypoint.…"   6 seconds ago   Up 5 seconds   0.0.0.0:8080->80/tcp, :::8080->80/tcp   wonderful_shockley
```

Still no luck:
```
mypc$ curl -v 192.168.50.254:8080
*   Trying 192.168.50.254:8080...
^C⏎                                                                                
```

## Is it the port mapping?

As you can see above, the port is mapped on all IPs, so that's not the problem.

Can I reach the nginx from the `dockerhost` itself? Let's see:
```
root@dockerhost:~# curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
root@dockerhost:~# 
```
That one looks good.

Maybe it's something between the `dockerhost` and `mypc`?

## Is it the router?

So how does the connection between the `dockerhost` and `mypc` look like? I'm
configuring the `dockerhost` over ssh, so it should be fine, but just to be sure:
```
mypc$ ping 192.168.50.254
PING 192.168.50.254 (192.168.50.254) 56(84) bytes of data.
64 bytes from 192.168.50.254: icmp_seq=1 ttl=63 time=0.764 ms
64 bytes from 192.168.50.254: icmp_seq=2 ttl=63 time=0.638 ms
64 bytes from 192.168.50.254: icmp_seq=3 ttl=63 time=0.628 ms
^C
--- 192.168.50.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2071ms
rtt min/avg/max/mdev = 0.628/0.676/0.764/0.061 ms
```

Maybe something is blocking this particular port? I'll kill the nginx container for a while and replace it with netcat on the `dockerhost`:
```
root@dockerhost:~# nc -l 8080
```
and connect to it from `mypc`:
```
mypc$ nc 192.168.50.254 8080
ping
pong
```
It works without any issues, I'm able to send and receive data.

## Is it the `dockerhost`?

Ok, what do we know so far? The packets are sent correctly from `mypc` and they are
able to reach the `dockerhost`. For non-docker workloads, the transmission works
fine. We are also able to reach the container from the localhost. What is the
issue then?

I'll inspect the traffic on the `dockerhost` itself. I'll run a tcpdump to
capture the traffic on all interfaces and run `curl localhost:8080`:
 ```
root@dockerhost:~# tcpdump -q -n -i any port 8080 or port 80
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
21:41:49.118701 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 0
21:41:49.118855 lo    In  IP 127.0.0.1.8080 > 127.0.0.1.40892: tcp 0
21:41:49.118891 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 0
21:41:49.119096 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 78
21:41:49.119213 lo    In  IP 127.0.0.1.8080 > 127.0.0.1.40892: tcp 0
21:41:49.119289 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.119302 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.119335 vethfdb534f P   IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 0
21:41:49.119335 docker0 In  IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 0
21:41:49.119379 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.119384 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.119698 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 78
21:41:49.119704 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 78
21:41:49.119733 vethfdb534f P   IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 0
21:41:49.119733 docker0 In  IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 0
21:41:49.119993 vethfdb534f P   IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 238
21:41:49.119993 docker0 In  IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 238
21:41:49.120024 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.120029 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.120076 vethfdb534f P   IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 615
21:41:49.120082 lo    In  IP 127.0.0.1.8080 > 127.0.0.1.40892: tcp 238
21:41:49.120076 docker0 In  IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 615
21:41:49.120098 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.120102 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.120110 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 0
21:41:49.120343 lo    In  IP 127.0.0.1.8080 > 127.0.0.1.40892: tcp 615
21:41:49.120364 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 0
21:41:49.121188 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 0
21:41:49.121558 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.121569 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.123905 vethfdb534f P   IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 0
21:41:49.123905 docker0 In  IP 172.17.0.2.80 > 172.17.0.1.35062: tcp 0
21:41:49.124004 docker0 Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.124012 vethfdb534f Out IP 172.17.0.1.35062 > 172.17.0.2.80: tcp 0
21:41:49.124627 lo    In  IP 127.0.0.1.8080 > 127.0.0.1.40892: tcp 0
21:41:49.124663 lo    In  IP 127.0.0.1.40892 > 127.0.0.1.8080: tcp 0
^C
36 packets captured
50 packets received by filter
0 packets dropped by kernel
root@dockerhost:~#
```
The flags used, are (see the [manual](https://www.tcpdump.org/manpages/tcpdump.1.html) for details):
```
-q - quiet output, suppress the details
-n - don't convert addresses to names
-i interface - select the interface, any can be used to select all
```
and the filter for ports 8080 or 80 is intended to capture the traffic coming from the *outside* (the left side of the port mapping of the docker container) and the traffic going into the container itself (the right side).

That obviously worked fine, but please do take a look. It's really cool to see
how the traffic flows through various interfaces and the `docker0` bridge!

Now, let's run the same tcpdump but launch the `curl` from `mypc`:
```
root@dockerhost:~# tcpdump -q -n -i any port 8080 or port 80
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
21:43:37.599268 enp2s0 In  IP 192.168.30.252.56084 > 192.168.50.254.8080: tcp 0
21:43:38.660223 enp2s0 In  IP 192.168.30.252.56084 > 192.168.50.254.8080: tcp 0
21:43:40.708210 enp2s0 In  IP 192.168.30.252.56084 > 192.168.50.254.8080: tcp 0
^C
3 packets captured
5 packets received by filter
0 packets dropped by kernel
root@dockerhost:~# 
```
Nothing. Well, almost nothing. The packets do come in, but then are
swallawed somewhere. The firewall? But shouldn't that work reasonably well out
of the box?

## Let's take a look at the firewall
It's time to look at the firewall. I know you would have done that sooner, but let's take a look together ;)

The ufw is disabled:
```
root@dockerhost:~# ufw status
Status: inactive
root@dockerhost:~#
```
and the iptables look like this:
```
root@dockerhost:~# iptables -L -v -n
# Warning: iptables-legacy tables present, use iptables-legacy to see them
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   57  3420 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
   57  3420 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
   57  3420 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   22  1320 ACCEPT     tcp  --  !docker0 docker0  0.0.0.0/0            172.17.0.2           tcp dpt:80

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
   57  3420 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   57  3420 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           
root@dockerhost:~#
```
and for the `nat` chain:
```
root@dockerhost:~# iptables -t nat -v -n -L
# Warning: iptables-legacy tables present, use iptables-legacy to see them
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   63  3852 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
   22  1320 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
root@dockerhost:~#
```
The policies are all `ACCEPT` as far as I can see, the `DNAT` is in place to direct the traffic to our container... I don't see anything obviously wrong at this point.

## Dear firewall, how should you be debugged?

There is a really good
[post](https://backreference.org/2010/06/11/iptables-debugging/) on debugging
the iptables, so I'm not going to explain that in detail, but in general it is
possible to use a `TRACE` target to log the packet journey through the iptables.

I don't want to trace all of the packets, because that will make the log
unreadable, so I'll have to be selective enough and at the same time apply the
TRACE target as soon as possible. The best fit for that is the `raw` table, that
is processed very soon on the packet arrival ([see the netfilter packet flow diagram](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)):
```
root@dockerhost:~# -A PREROUTING -p tcp -m tcp --dport 8080 -j TRACE
```
The logs can be found in the `/var/log/kern.log` or simply in the `dmesg`.

What can we see for the `curl` run from `mypc`?
```
# redacted a bit for readability
[ 1288.683905] TRACE: raw:PREROUTING:policy:1 IN=enp2s0 OUT= MAC=... SRC=192.168.30.252 DST=192.168.50.254 ...
[ 1288.684048] TRACE: filter:FORWARD:rule:1 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
[ 1288.684079] TRACE: filter:DOCKER-USER:return:1 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
[ 1288.684104] TRACE: filter:FORWARD:rule:2 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
[ 1288.684130] TRACE: filter:DOCKER-ISOLATION-STAGE-1:return:2 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
[ 1288.684158] TRACE: filter:FORWARD:rule:4 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
[ 1288.684182] TRACE: filter:DOCKER:return:1 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
[ 1288.684206] TRACE: filter:FORWARD:policy:7 IN=enp2s0 OUT=docker0 MAC=... SRC=192.168.30.252 DST=172.17.0.2 ...
```
Ok, the packet goes in, hits the `raw` table with our `TRACE` target and then moves to the `filter` table to go through the `FORWARD` and `DOCKER` chains. It ends up using the `FORWARD` chain policy of the `filter` table. Can you find it above?

There is something wrong here though... Please take a look at the first line:
```
... raw:PREROUTING:policy:1 ...
```
It says, that the packet goes through the `raw` table, `PREROUTING` chain and uses the policy which is the rule number 1 (the rule number for the policy will be simply +1 of the last rule number of the chain). How can it be 1 here if we have just added a rule to that chain (the `TRACE` one)?

```
root@dockerhost:~# iptables -t raw -L
# Warning: iptables-legacy tables present, use iptables-legacy to see them
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
TRACE      tcp  --  anywhere             anywhere             tcp dpt:http-alt
```

Oh, stop complaining about those `iptables-legacy` already... Wait, what?
```
root@dockerhost:~# iptables-legacy -t raw -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination        
```
Interesting!

How about the `filter` table?
```
root@dockerhost:~# iptables-legacy -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy DROP)
target     prot opt source               destination         
DOCKER-USER  all  --  anywhere             anywhere            
DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain DOCKER (1 references)
target     prot opt source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination         
DOCKER-ISOLATION-STAGE-2  all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination         
DROP       all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            

Chain DOCKER-USER (1 references)
target     prot opt source               destination         
RETURN     all  --  anywhere             anywhere            
root@dockerhost:~#
```

Can you see that `DROP` policy in the `FORWARD` chain? I'll change it to `ACCEPT`:
```
root@dockerhost:~# iptables-legacy -P FORWARD ACCEPT
```
and retry:
```
mypc$ curl -v 192.168.50.254:8080
*   Trying 192.168.50.254:8080...
* Connected to 192.168.50.254 (192.168.50.254) port 8080 (#0)
> GET / HTTP/1.1
> Host: 192.168.50.254:8080
> User-Agent: curl/7.84.0
> Accept: */*
> 
...
```
It works!

## What happened?

It seems that the docker snap uses legacy iptables to configure the firewall,
whereas the one installed with APT configured it with the currently used
nftables. I haven't been able to quickly find a good explanation of how iptables
and nftables work when combined, only advices not to configure the rules for
both at the same time.

How to fix it? Well, have you tried forcing an (un)expected reboot? :) This will
cleanup the legacy iptables leftovers made by the docker snap and leave only the
nftables filters.

## Summary
In this post we went together through debugging a seemingly trivial issue with
the docker networking.  While the root cause is not something that I would
expect to happen on any of the production machines, I believe that it was a
great educational excercise and an opportunity to look closer into the Linux
firewall and the tools that might help you troubleshoot connectivity issues in
the future.

Thank you for reading and have a great day!
