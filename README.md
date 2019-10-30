# run-dfz

<p align="center">
  <img width="40%" height="40%" src="run-dfz.png">
</p>

One of the things I have wanted to test over the years was having realistic Internet BGP data inside of a lab network. Traditionally this would entail peering with a real router in production. I have never been in an environment that would be OK with this, so I always just simulated the Internet with a couple of prefixes. This fixes that problem and allows you to run a router and get hundreds of thousands of BGP prefixes into your lab environment.


# Architecture

This works by running a guestshell instance on a CSR1000v router and then peering from the guestshell to the host router itself. The guestshell is injecting the prefixes in, and then you have a "real router" to then peer elsewhere in your lab network.

![Architecture](run-dfz-arch.png)

# Installation

1. Provision a CSR1000v in your lab emulation software with a fair amount of memory (I tested with 16GB, but you may be able to get away with less).
2. Connect the router to the outside world via a cloud/NAT/bridged connection. You will need to pull the necessary files onto the router.

    ![Lab Topology](run-dfz-topology.png)

3. Provision the router with guestshell and the necessary bits to allow front-panel port connectivity (and Internet access) to guestshell.
    ```
    conf t
    interface VirtualPortGroup0
    description *** SVI facing guestshell container ***
    ip nat inside
    ip address 192.168.100.1 255.255.255.0
    no shut
    !
    interface GigabitEthernet1
    description *** Outside Network Connectivity ***
    ip nat outside
    ip address 192.168.4.96 255.255.255.0
    no shut
    !
    ip route 0.0.0.0 0.0.0.0 192.168.4.1
    !
    access-list 1 permit 192.168.100.0 0.0.0.255
    !
    ip nat inside source list 1 interface GigabitEthernet1 overload
    !
    app-hosting appid guestshell
    resource profile custom cpu 20000 memory 4096
    !
    iox
    !
    app-hosting appid guestshell
    resource profile custom
    resource profile custom cpu 20000
    resource profile custom cpu 20000 memory 4096
    !
    end
    !
    guestshell enable virtualPortGroup 0 guest-ip 192.168.100.2 name-server 8.8.8.8
    !
    ```
4. Enter the guestshell.

    `guestshell`

5. Install tmux and git and upgrade a couple packages.

    `sudo yum install tmux git -y`

    `sudo yum update -y nss curl libcurl`

6. Clone this git repo inside of your guestshell and enter the directory.

    `git clone https://github.com/clay584/run-dfz.git /bootflash/run-dfz && cd /bootflash/run-dfz`

7. Extract the routes files to the router's bootflash (bootflash has plenty of space available).

    `tar zxvf routes.tar.gz`

8. Enter a tmux shell.

    `tmux`

9. Run bgpsimple and point to a routes file.

    `sudo ./bgp_simple -myas 64999 -myip 192.168.100.2 -peerip 192.168.100.1 -peeras 65000 -p routes-short.txt -holdtime 300 -keepalive 30`

10. Exit the guestshell

    `ctrl + b` then press `d` to disconnect from the tmux session. Then type `exit` to leave guestshell and go back to the router prompt.

11. Configure BGP on the router.
    ```
    conf t
    router bgp 65000
    bgp log-neighbor-changes
    no bgp default ipv4-unicast
    neighbor 192.168.100.2 remote-as 64999
    !
    address-family ipv4
    neighbor 192.168.100.2 activate
    exit-address-family
    !
    end
    ```
12. The BGP peering should come up and you should get a lot of routes in the RIB.

    ```
    Router#sh ip bgp summary
    BGP router identifier 192.168.100.1, local AS number 65000
    BGP table version is 1, main routing table version 1
    2630 network entries using 652240 bytes of memory
    2630 path entries using 357680 bytes of memory
    232/0 BGP path/bestpath attribute entries using 64960 bytes of memory
    205 BGP AS-PATH entries using 9830 bytes of memory
    6 BGP community entries using 144 bytes of memory
    0 BGP route-map cache entries using 0 bytes of memory
    0 BGP filter-list cache entries using 0 bytes of memory
    BGP using 1084854 total bytes of memory
    BGP activity 2630/0 prefixes, 2630/0 paths, scan interval 60 secs

    Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
    192.168.100.2   4        64999    2632       2        1    0    0 00:00:08     2630
    ```

13. Once you have this working, it is ok to disconnect the router from the real network. It is no longer needed after installation.

# Route Files

Inside of the routes.tar.gz file, there are four files. You can use any of these you want. I found that 800k prefixes tends to crash the CSR1000v routers in my lab, so that's why I created the partial files.

* `routes.txt` - Full tables (~800k prefixes) from one peer at the Equinix Miami MI1 data center (RIPE RIS Data from 10/28/2019).
* `routes-short.txt` - Partial tables (~175k prefixes) from one peer at the Equinix Miami MI1 data center (RIPE RIS Data from 10/28/2019).
* `routes2.txt` - Full tables (~800k prefixes) from a second peer at the Equinix Miami MI1 data center (RIPE RIS Data from 10/28/2019).
* `routes2-short.txt` - Partial tables (~175k prefixes) from a second peer at the Equinix Miami MI1 data center (RIPE RIS Data from 10/28/2019).
