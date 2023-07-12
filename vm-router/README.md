# VM Router

This page provides general steps to set up a virtual router for managing VMs' traffic in VMWare Workstation and VirtualBox. The specific goals I aimed to accomplish are as follows:

- Restrict VMs from accessing the internet directly and  instead through the virtual router (and ultimately through my VPN provider's servers).
- Prevent VMs from accessing devices on my local network.
- Optional: Isolate VMs from each other, preventing them from communicating.

## Main Idea

1. Create a LAN network in VM Ware or VirtualBox (it’s called LAN Segment in VM Ware and internal network in VirtualBox) and put all machines (virtual router and VMs you want handled by it) in it. This way all machines will have an ethernet adapter through which they can communicate (as if they were connected through a switch). This adapter does not provide internet access.

2. Give the virtual router an additional NAT network adapter so that it can access the internet through it.

3. Configure the virtual router to route all of its traffic through a VPN provider’s servers and to work as a router for the other VMs.

The following sections describe different methods I used to achieve this.

## 1. [pfSense](https://en.wikipedia.org/wiki/PfSense) Virtual Router (slow?)

pfSense is a predominant router OS used in many business and non-business environments. This is probably the most straightforward way to do this assuming your provider has documented compatibility with pfSense. In my case, Nord has [detailed instructions about how to set this up](https://support.nordvpn.com/Connectivity/Router/1626958942/pfSense-2-5-Setup-with-NordVPN.htm).

In short, I just had to install and configure the [pfSense ISO](https://www.pfsense.org/download/) in a VM and configure it to route its traffic to my VPN provider’s servers. This worked well and it is nice that pfSense allows you to add a lot of firewall and intrusion prevention rules, however, this performed very slow for me. My average speeds on a single VM client were around 8 times slower than the speed I’m able to get by just connecting my main computer to the same VPN server. However, I might have done a configuration error. This is potentially the easiest and most practical solution for most people since, after installing the ISO, you can simply configure pfSense as if it was a regular router (with a ton of features by the way).

### 1.1 Steps

1. Create a LAN Network in VM Ware (LAN Segment) or VirtualBox (internal network).

2. [Setup a pfSense VM](https://docs.netgate.com/pfsense/en/latest/), make sure to add to it the previously created adapter as well as a NAT one. Give the machine enough computational and memory resources. Experiment with these parameters for best performance.

3. Setup the pfSense VM to connect to the internet through a VPN.

4. Give your main virtual machines access only to the LAN Network adapter you created.

Once the router is configured, this should work without any DHCP/manual configurations on the main machines and they should only be able to access the internet and local network through the virtual router.

## 2. [Alpine](https://en.wikipedia.org/wiki/Alpine_Linux) Virtual Router (best overall)

Alpine is a very lightweight and secure Linux distribution used frequently in business environments. Therefore, I decided to use it and set it up as a virtual router. In short, you just need to setup the [ISO](https://www.alpinelinux.org/downloads/) on a VM and configure as desired. One can configure a general purpose OS like Alpine to handled network traffic pretty much in any way desired. However, to keep things simple for this tutorial, I just set it up to work as a DHCP server and NAT router for my other VMs.

### 2.1 Steps

1. Create a LAN Network in VM Ware (LAN Segment) or VirtualBox (internal network).

2. Setup an Alpine VM, make sure to add to it the previously created adapter as well as a NAT one. Give the machine enough computational and memory resources. Experiment with these parameters for best performance.

3. To set up the Alpine VM as a DHCP server and configure it as a router for the other VMs, follow the steps outlined in [this webpage](https://cylab.be/blog/221/a-light-nat-router-and-dhcp-server-with-alpine-linux). The instructions cover setting up the DHCP server and configuring the VM as a NAT router. Before configuring the NAT forward rule, ensure that the machine is set up to connect to your preferred VPN upon boot. Once that is done, you can proceed to set up the NAT forward rule for the respective VPN's network interface.

4. Give your main virtual machines access only to the LAN Network adapter you created.

Once this setup is complete, the remaining VMs with access to the router should be able to obtain IP addresses via DHCP and access the internet through the router VM. Upon configuring and testing this setup, I found that the performance was nearly identical to directly connecting my physical machine to my VPN provider's servers.

### 2.2 Comments

In my case, the router VM occupied less than 150 MB of space in my machine. This is relatively little so it might be worth it to have different such routers for different machines/networks instead of configuring one router that handles all of those.

## 3. [Gluetun](https://github.com/qdm12/gluetun) Proxy (my preferred method)

### 3.1 Idea

I decided to try to connect my VMs to the internet through a Gluetun container somehow because it is a very fast and easy way to connect LAN devices to various VPN providers. I decided to use Alpine Linux as the host OS for my Gluetun service because it is relatively lightweight and simple to use. The only problem I had left was that Gluetun allows you to connect devices to it through a proxy (HTTP or SHADOWSOCKS), so after setting it up I had to configure my main virtual machines to route all of their traffic through the Gluetun proxy. To do this, I used [tun2socks](https://github.com/xjasonlyu/tun2socks) on the machines to create a tun network interface that allowed them to route traffic through the proxy. After setting this up, with pretty much all configurations, the performance of a single VM client was almost identical (less than 5 Mbps slower) to the one of my main computer connecting to the same VPN server.

### 3.2 Speed and Considerations

#### 3.2.1 HTTP

Setting up Gluetun as a HTTP proxy makes for a fast choice because traffic between the client and Gluetun is not encrypted. However, this prevents your clients to use UDP by default. If you need to use UDP and would like to use this configuration you will have to implement UDP over TCP somehow. In my case I only found practical issues with DNS resolution, which I solved by configuring DNS over TLS on the machine (Linux, systemd-resolved). It should also be possible to solve the DNS issues by configuring DNS over HTTPS on the machine. If you want the traffic between the router and the clients to be encrypted (you don’t want your virtual machines to sniff each other’s traffic), the SHADOWSOCKS configuration might be better for you (even cheap physical routers provide encryption options, like CCMP).

#### 3.2.2 SHADOWSOCKS

This configuration fully supports UDP and TCP allowing clients to route all their traffic to the VPN servers seemingless. I tried all currently available cipher configurations (chacha20-ietf-poly1305, aes-256-gcm,  aes-128-gcm) and the results were for the most part similar, the slowest speed was obtained with the aes-256-gcm configuration but I did not test this rigorously (speedtest.net, multiple runs). The slowest speed was only 5 Mbps slower than my main computer’s max speed over the same VPN server. There was a very small performance disadvantage as compared with the HTTP configuration, so, for most practical uses, the default cipher (chacha20-ietf-poly1305) should be sufficient and performant.

### 3.3 Steps

1. Create a LAN Network in VM Ware (LAN Segment) or VirtualBox (internal network).

2. Setup an Alpine VM (for the virtual router), make sure to add to it the previously created adapter as well as a NAT one. Give the machine enough computational and memory resources. Experiment with these parameters for best performance.

3. Configure Docker and Gluetun on the virtual router machine.

    - Un-comment the community path in `/etc/apk/repositories`
    - Follow the steps [here](https://wiki.alpinelinux.org/wiki/Docker) to enable Docker.
    - Refer to the [Gluetun wiki](https://github.com/qdm12/gluetun/wiki) to start a container with respect to your VPN service provider.

4. Setup the LAN Network interface, so that the router has an IP and subnet on that interface at boot. This can be done by adding the following to `/etc/network/interfaces` and running the following command:

    ```{txt}
    auto <lan-interface-name-here>
    iface <lan-interface-name-here> inet static
        address 192.168.0.1
        netmask 255.255.255.0
    ```

    Bring the interface up:

    ```{bash}
    sudo ifup <lan-interface-name-here>
    ```

5. Setup the DHCP server so that machines connected to the LAN Net virtual switch can obtain an IP and information about the subnet.

    ```{bash}
    sudo apk add dhcp
    ```

    Specify the LAN interface's network, range of IPs to provide, and network details to provide to the VMs in `/etc/dhcp/dhcp.conf`:

    ```{txt}
    subnet 192.168.0.0 netmask 255.255.255.0 {
        range 192.168.0.10 192.168.0.20;
        option subnet-mask 255.255.255.0;
    }
    ```

    Enable the DHCP service at boot

    ```{bash}
    sudo rc-update add dhcpd
    sudo rc-service dhcpd start
    ```

6. Configure the other VMs. For initial setup, it might be better to give them access to a NAT adapter so that they can access the internet and can download tun2socks. Once they have tun2socks, that adapter can be removed and just have the LAN Network adapter (or any others depending on what you want to achieve).

7. On the client machines, use tun2socks to create a network adapter that can route traffic to the Gluetun proxy. That can be done with one command:

    ```{bash}
    tun2socks -device tun://<new interface name> -proxy <protocol here>://<router ip>:<gluetun port>
    ```

8. Set up client machines to route all traffic through the tun2socks interface. This is OS specific but the process should be similar. This can be done in Linux as follows:

    - Assign an IP address to the tun2socks interface and specify the corresponding subnet:

        `sudo ip addr add <new interface IP>/<MASK> dev <tun2socks interface name here>`

    - Activate interface

        `sudo ip link set <tun2socks interface name here> up`

    - Delete the default gateway (if there’s one) and use the new interface instead

        ```{bash}
        sudo ip route del default
        sudo ip route add default via <new interface IP here> dev <interface name here>
        ```

9. Setup the VMs to do this setup automatically at boot. There are many ways to do this and this is OS specific. In Linux, consider setting a service (managed by systemd usually) that handles this.

I like this setup because it offers convenient management of various VPN configurations through Gluetun. Moreover, the advantage of Gluetun being a container allows for instant creation of multiple instances with different setups. In the future, it would be good to have a container router that has all the features of Gluetun while incorporating additional ones to function as a standard and efficient router.

## Comments and Security Considerations

Considerations have been taken to isolate the virtual machines from the local network, however, these setups have not been tested rigorously, please make sure to test and modify these setups according to your network security requirements.

With some edits these setups could do other interesting things. For example, if given a bridge adapter to the local network, the virtual router could route traffic from the physical devices on the network. Additional configuration on the clients might be needed.

I'm not a networking expert but setting this up helped me learn a lot about it. I hope this helps you too.
