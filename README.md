# Dual-LND-Hybrid-VPS
This guide is a simpler how-to for implementing Hybrid Mode for 2+ LND Nodes via VPS. The original guide can be found [here](https://github.com/TrezorHannes/vps-lnbits). When finished, you'll be able to run one or more of your Lightning Nodes via Tor and obfuscate your Clearnet IP Adress via a paid VPS

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/d/d5/Alter_Elbtunnel_%28St._Pauli%29.jpg/2880px-Alter_Elbtunnel_%28St._Pauli%29.jpg" alt="Alter Elbtunnel (erbaut 1911) – Wikipedia" width="320" height="240">

Here's my current setup shared with you, and your intend can be manyfold, you may
- have a dynamic IP from your Internet Service Provider
- want to hide your home IP from the world, for whatever reason
- desire to decrease your Lightning Node HTLC Routing times, so instead of running Tor only, you want Clearnet availability, too
- run more than one Lightning Node, and want both to leverage the same VPN tunnel
- are just curious and want to tinker around a bit, because it's good to have those skills when demand for experience continues to rise

## Table of Content

- [Pre-Amble](#pre-amble)
  - [Objective](#objective)
  - [Challenge](#challenge)
  - [Proposed Solution](#proposed-solution)
- [Pre-Reads](#pre-reads)
- [Pre-Requisites](#pre-requisites)
- [Preperations](#preperations)
  - [Make notes](#make-notes)
  - [Visualize](#visualize)
  - [Secure](#secure)
- [Let's get started (LFG!)](#lets-get-started-lfg)
  - [1) Lightning Node](#1-lightning-node)
  - [2) VPS: Setup](#2-vps-setup)
  - [3) VPS: Connect to your VPS and tighten it up](#3-vps-connect-to-your-vps-and-tighten-it-up)
  - [4) VPS: Install OpenVPN Server](#4-vps-install-openvpn-server)
  - [5) VPS: Retrieve the OpenVPN config & certificate](#5-vps-retrieve-the-openvpn-config--certificate)
- [Into the Tunnel](#into-the-tunnel)
  - [6) LND Node: Install and test the VPN Tunnel](#6-lnd-node-install-and-test-the-vpn-tunnel)
  - [7) VPS: Add routing tables configuration into your droplet docker](#7-vps-add-routing-tables-configuration-into-your-droplet-docker)
  - [8) LND Node: LND adjustments to listen and channel via VPS VPN Tunnel](#8-lnd-node-lnd-adjustments-to-listen-and-channel-via-vps-vpn-tunnel)
  - [9) LND Node: Start your VPN Client again](#9-lnd-node-start-your-vpn-client-again)
  - [10) Celebrate and wrapper](#10-celebrate-and-wrapper)
- [Appendix & FAQ](#appendix--faq)


## Pre-Amble

### Objective
Your own, non-custodial [Lightning-Network](https://github.com/lightningnetwork/lnd) Node(s) running on both Tor and Clearnet [(Hybrid-Mode)](https://github.com/blckbx/lnd-hybrid-mode) installed on a cheap, but anonymous [Virtual Private Server (VPS)](https://www.webcentral.com.au/blog/what-does-vps-stand-for). This guide will outline the scenario for **two nodes**, so in case you only have one, or more than two, adjust the appropriate steps below.

### Challenge
We want payment options with ₿itcoin to be fast, reliable, non-custodial - but the service should ideally not be easily to be identifiable. We also see a lot of reliance on Tor to provide that, which sometimes comes with trade-offs in speed and reliability.

### Proposed Solution
There are plenty of ways how to solve for this. This creates hesitance to implement, especially when you're not very technical. This guide is supposed to provide _one approach_, whilst there remain many other ways to Rome. 
Take your time following this through. It might take you 1hr, depending highly on your skill. So don't go in here in a rush.


## Pre-Reads
This guide heavily relies on the intelligence and documentation of others 🙏, but putting those together to one picture creates the last 10% hurdle which is sometimes the highest. Have a careful read through the following articles, to get a deeper understanding on some of the lighter references we'll be using further below
- [Hybrid-Mode for LND](https://github.com/blckbx/lnd-hybrid-mode)
- [TURN YOUR SELF HOSTED LIGHTNING NETWORK NODE TO PUBLIC IN 10 MINUTES](https://www.mobycrypt.com/turn-your-self-hosted-lightning-network-node-to-public-in-10-minutes/)
- [OpenVPN for Docker](https://github.com/kylemanna/docker-openvpn)


## Pre-Requisites
- running `lnd-0.14.2-beta` or later. This can either be [Umbrel](https://getumbrel.com), [Raspiblitz](https://github.com/rootzoll/raspiblitz), [MyNode](https://mynodebtc.com/) or even a bare [RaspiBolt](https://raspibolt.org/)
- Technical curiosity and not too shy to use the command-line
- An SSH connection to your node, and to the VPS as well. On Windows, use something like [putty](https://www.putty.org/) and get [putty-gen](https://www.ssh.com/academy/ssh/putty/windows/puttygen), too
- VPS Account at DigitalOcean or any alternative VPS Solution out there offering similar capabilities (it's critical they offer a public IP for you)

[![DigitalOcean Referral Badge](https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%201.svg)](https://www.digitalocean.com/?refcode=5742b053ef6d&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)

_Disclaimer: this is a ref link, gets you $100 in credit over 60 days, while the cheapest option we use here comes at a cost of $5/month._


## Preperations
The better we prepare, the more we can deal with blindspots and the unexpected.

### Make notes
It's generally advised to document your own steps. Make a bucket-list of things you've done, and a ToDo to go through in case your environment changes. Imagine yourself 18 months from now, you want to setup this new hardware-node: Will you remember all the steps or extra corners you've taken?
Suggested Laundry-List, you can tick them off while you go through this guide
- [ ] IP-Adresses of VPS external, VPS Tunnel, Node Tunnel
- [ ] Ports which needs forwarding
- [ ] ToDos
- [ ] Questions / open items

### Visualize
Some of us are visual people. Draw your diagram to get an idea how you want things to flow
![Hight-lvl-Flowchart](https://github.com/TrezorHannes/Dual-LND-Hybrid-VPS/blob/main/Hybrid-LND-VPN.drawio.png?raw=true)

### Secure
It goes without saying, but this guide doesn't go into the necessary security steps in detail, and can't take on liability for any things breaking or losing funds. Ensure you don't get reckless, start with small funds you're ok to lose. Keep an eye on developments or in touch with the active Telegram Groups, to get news and updates with low delays. Also, would recommend to do those steps with a peer, so you follow a second pair of eye review. Lastly, 2fa / yubikeys are your friends!


## Let's get started (LFG!)
Well, let's get into it, shall we?!

### 1) Lightning Node
In this guide, we will consider you have two **Lightning Nodes up and running**, connected via Tor and some funds on it. You also have SSH access to both and administrative privilidges.

### 2) VPS: Setup 
In case you don't have a **VPS provider** already, sign-up with [my referal](https://m.do.co/c/5742b053ef6d) or [pick another](https://www.vpsbenchmarks.com/best_vps/2022) which provides you with a static IP and cheap costs. Maybe you even prefer one payable with Lightning ⚡, like [LunaNode](https://www.lunanode.com/). In case you go for DigitalOcean, here are the steps to create a Droplet, shouldn't take longer than a few minutes:
   - [ ] add a new Droplet on the left hand navigation
   - [ ] chose an OS of your preference, I have Ubuntu 20.04 (LTS) x64
   - [ ] take the Basic Plan with a shared CPU, that's enough power. You can upgrade anytime if necessary
   - [ ] Switch the CPU option to "Regular Intel with SSD", which should get you down to $5/month
   - [ ] You don't need an extra volume, but pick a datacenter region of your liking
   - [ ] Authentication: Chose the SSH keys option and follow the next steps to add your public keys in here for secure access. For Windows, with putty and putty-gen referenced above, you should be relatively quick to use those keys instead of a password. [For Linux users](https://serverpilot.io/docs/how-to-use-ssh-public-key-authentication/), you probably know your ways already.
   - [ ] Add backups (costs), Monitoring or IPv6 if you wish to, however this guide won't use any of those items
   - [ ] Lastly, chose a tacky hostname, something which resonates with you, eg myLNBits-VPS

After a few magic cloud things happening, you have your Droplet initiated and it provides you with a public IPv4 Adress. Add it to your notes! In this guide, I'll refer to it as `VPS Public IP: 207.154.241.101`


### 3) VPS: Connect to your VPS and tighten it up
Connect to your VPS via `SSH root@207.154.241.101` and you will be welcomed on your new, remote server. Next steps are critical to do right away, harden your setup:
   - [ ] Update your packages: `apt-get update` and `apt-get upgrade`
   - [ ] Install Docker & tmux: `apt-get install docker.io tmux`
   - [ ] Enable Docker automated start: `systemctl start docker.service`
   - [ ] Enable Uncomplicated Firewall (UFW) and add ports to be allowed to connected to: 
```
$ apt install ufw
$ ufw default deny incoming
$ ufw default allow outgoing
$ ufw allow OpenSSH
$ ufw allow 9735 comment 'LND Main Node 1'
$ ufw allow 9736 comment 'LND Main Node 2'
$ ufw enable
```
   - [ ] Follow [further hardening steps](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04), eg setting up non-root users for additional security enhancements.
   - [ ] Install fail2ban to protect your SSH user, it runs automatically on it's own `sudo apt install fail2ban`

### 4) VPS: Install OpenVPN Server
Now we will get OpenVPN installed, but using a Docker Setup like [Krypto4narchista](https://twitter.com/_pthomann) suggests [here](https://www.mobycrypt.com/turn-your-self-hosted-lightning-network-node-to-public-in-10-minutes/). It's easier to setup, but needs some tinkering with port forwarding, which we will go into in a bit.
   - [ ] `export OVPN_DATA="ovpn-data"` which sets a global-name placeholder for your VPN to be used for all the following commands. You can make this permanent by adding this to survive any reboot via `nano .bashrc`, add it to the very bottom => CTRL-X => Yes. 
   - [ ] `docker volume create --name $OVPN_DATA` notice how the $ indicates picking up the placeholder you have defined above
   - [ ] `docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://207.154.241.101`, whereby you need to adjust the 207.154.241.101 with your own **VPS Public IP**.
   - [ ] `docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki` this generates the necessary VPN certificate password. Take your password manager and create a secure pwd, which you will store safely. It will be needed once we create client-configuration files for your node to connect later.
   - [ ] `docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp -p 9735:9735 -p 9735:9735/udp -p 9736:9736 -p 9735:9735/udp --cap-add=NET_ADMIN kylemanna/openvpn` this works under two assumptions. If any of those aren't true, you need to adjust your settings, either on your node, or by starting the docker container with different ports: 
     1) your current LND Node 1 configuration is listening on port 9735, which you can verify by looking into your `cat ~/.lnd/lnd.conf` => `[Application Options]` => `listen=0.0.0.0:9735`
     2) your second LND Node 2 configuration is listening on port 9736, which you can verify by looking into your `cat ~/.lnd/lnd.conf` => `[Application Options]` => `listen=0.0.0.0:9736`

Your OpenVPN Server is now running, which means the Internet can now connect to your VPS via ports 9735 and 9736 (and 22 SSH of course), and it has a closed tunnel established on port 1194. You need to complement your notes with the IP-Adresses which are essentially added with the running server.

   - [ ] CONTAINER-ID: `docker ps` to list your docker container. In the first column, you will find the `CONTAINER-ID`, usually a cryptic 12-digit number/character combination. Copy into the clipboard and make a note of it. 
   - [ ] Docker Shell: Get into the container, with `docker exec -it <CONTAINER-ID> sh`. 
   - [ ] VPS Docker IP: Run `ifconfig` and you typically find 3 devices listed with IPs assigned. Make a note of the one with eth0, which is your own `VPS Docker IP: 172.17.0.2`. Type `exit` to get out of the docker-shell.

### 5) VPS: Retrieve the OpenVPN config & certificate
In this section we'll switch our work from setting up the server towards getting your LND node ready to connect to the tunnel. For this, we will retrieve and transfer the configuration files from your VPS to your nodes 1 and 2.
   - [ ] `docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full NODE-NAME1 nopass` whereby `NODE-NAME1` should be changed to a unique identifier you chose. For example, if your LND Node is called "BringMeSomeSats", I suggest to use that - with all lowercase.
   - [ ] `docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full NODE-NAME2 nopass` whereby `NODE-NAME2` should be changed similar for LND Node 2.
   - [ ] `docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient NODE-NAME1 > NODE-NAME1.ovpn` which will prompt you to provide the secure password you have generated earlier. Afterwards, it'll store `bringmesomesats.ovpn` in the directory you currently are.
   - [ ] Do the same for your Node 2: `docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient NODE-NAME2 > NODE-NAME2.ovpn`


## Into the Tunnel
We have installed the tunnel through the mountain, but need to get our LND Node to use it.

### 6) LND Node: Install and test the VPN Tunnel
Now switch to another terminal window, and SSH into your **Lightning Node 1**. We want to connect to the VPS and retrieve the VPN-Config file, to be able to establish the tunnel. Repeat the step for **Lightning Node 2**, with the ovpn-configuration file name adjusted.
```
$ cd ~
$ mkdir VPNcert
$ scp user@207.154.241.101:/home/user/bringmesomesats.ovpn /home/admin/VPNcert/
$ chmod 600 /home/admin/VPNcert/bringmesomesats.ovpn
```
_Note: You need to adjust `user`, the **VPS Public IP** and the absolute directory where the ovpn file is stored. Also make sure you are concious where you place this file on your LND Node, because we need it later._

So we have config file, secured in a directory, now we need to install OpenVPN & Tmux, start it up to see if it works. Tmux is a program allowing you to run services in the background. So even once you close the terminal, your OpenVPN Client will continue to run the tunnel. Read about [Tmux short-cuts here](https://tmuxcheatsheet.com/), or use alternatives like [screen](https://linuxize.com/post/how-to-use-linux-screen/).

**Important Warning**: Depending on your network-setup, there is a slight chance your LND Node Service gets interrupted. Be aware there might be small down-times of your lightning node, as we will reconfigure things. Be patient!

```
$ sudo apt-get install openvpn tmux
$ tmux new -s vps
$ sudo openvpn --config /home/admin/VPNcert/bringmesomesats.ovpn
```
You should see something similiar to the following output. Note this one line indicating the next important IP Adress `VPN Client IP: 192.168.255.6`. Make a note of it, we need it for port-configuration at the server, soon.
```
2022-03-02 17:38:10 library versions: OpenSSL 1.1.1k  25 Mar 2021, LZO 2.10
2022-03-02 17:38:10 TCP/UDP: Preserving recently used remote address: [AF_INET]207.154.241.101:1194
2022-03-02 17:38:10 UDP link local: (not bound)
2022-03-02 17:38:10 UDP link remote: [AF_INET]207.154.241.101:1194
2022-03-02 17:38:11 WARNING: 'link-mtu' is used inconsistently, local='link-mtu 1541', remote='link-mtu 1542'
2022-03-02 17:38:11 WARNING: 'comp-lzo' is present in remote config but missing in local config, remote='comp-lzo'
2022-03-02 17:38:11 [207.154.241.101] Peer Connection Initiated with [AF_INET]207.154.241.101:1194
2022-03-02 17:38:12 Options error: Unrecognized option or missing or extra parameter(s) in [PUSH-OPTIONS]:1: block-outside-dns (2.5.1)
2022-03-02 17:38:12 TUN/TAP device tun0 opened
2022-03-02 17:38:12 net_iface_mtu_set: mtu 1500 for tun0
2022-03-02 17:38:12 net_iface_up: set tun0 up
2022-03-02 17:38:12 net_addr_ptp_v4_add: 192.168.255.6 peer 192.168.255.5 dev tun0
2022-03-02 17:38:12 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
2022-03-02 17:38:12 Initialization Sequence Completed
```
The tunnel between your LND Node and your VPS VPN is established. We can place the VPN client into the background with `CTRL-B + CTRL-D`, which need to be pressed quickly after each other. You'll be out of the tmux session now, if you are curious if things still run, do `tmux a -t vps`, and do `CTRL-B + CTRL-D` to detach it again.

You can safely repeat this step at the second LND Node 2 now. The established tunnel won't effect your current running operations, and you need the tunnel IP of your second node for the next step for the port-forwarding table. If we don't know the IP, how should we tell the VPS where each package should go?


### 7) VPS: Add routing tables configuration into your droplet docker
Back to your terminal window connected to your VPS. We have the `VPN Client Node 1 IP: 192.168.255.6` and `VPN Client Node 2 IP: 192.168.255.11` now, which we need to tell our VPS where it should route those packets to. To achieve that, we'll get back into the docker container and add IPTables rules.

   - [ ] [Remember](#4-vps-install-openvpn-server) how to get into the container? Arrow-up on your keyboard, or do `docker ps` and `docker exec -it <CONTAINER-ID> sh`
   - [ ] Doublecheck your VPN Client IP, and adjust it in the following IPtables commands you enter into the container and confirm with Enter

```
$ iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9735 -j DNAT --to 192.168.255.6:9735
$ iptables -A PREROUTING -t nat -i eth0 -p udp -m udp --dport 9735 -j DNAT --to 192.168.255.6:9735
$ iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9736 -j DNAT --to 192.168.255.11:9736
$ iptables -A PREROUTING -t nat -i eth0 -p udp -m udp --dport 9736 -j DNAT --to 192.168.255.11:9736
$ iptables -t nat -A POSTROUTING -d 192.168.255.0/24 -o tun0 -j MASQUERADE
$ exit
```
What we basically do here, is assign a ruleset to say: As soon a packet arrives at device `eth0` on `port 9735/udp` and `/tcp`, forward it to the `VPN client` at `192.168.255.6:9735`, the same for `9736`, and vice versa everything at device `tun0`. If you have different ports or IPs, please make adjustments accordingly. 

### 8) LND Node: LND adjustments to listen and channel via VPS VPN Tunnel
We switch Terminal windows again, going back to your LND Node. A quick disclaimer again, since we are fortunate enough to have plenty of good LND node solutions out there, we cannot cater for every configuration out there. Feel free to leave comments or log issues if you get stuck for your node, we'll be looking at the two most different setups here. But this should work very similar on _MyNode_, _Raspibolt_ or _Citadel_.

**Caution:** If your second name is "Dangerously", do the same with the second LND Node now. But I would strongly suggest to finish running LND with Main Node 1 first, before you repeat this step at Node 2 just with the different port of `9736`.

Also, be **very** cautious with your `lnd.conf`. Make a backup before with `cp /mnt/hdd/lnd/lnd.conf /mnt/hdd/lnd/lnd.bak` so you can revert back when things don't work out. 
The brackets below indicate the section where each line needs to be added to. Don't place anything anywhere else, as it will prevent your LND from starting properly.

_Adjust ports and IPs accordingly!_

<details><summary>Click here to expand Raspiblitz / Raspibolt settings</summary>
<p>

   LND.conf adjustments, open with `sudo nano /mnt/hdd/lnd/lnd.conf`

[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735`           | # to add your VPS Public-IP |
   | `nat=false`                                 | # deactivate NAT |

[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                           | # ensure Tor is active |
   | `tor.v3=true`                               | # with the latest version. v2 is going to be deprecated this summer |
   | `tor.streamisolation=false`                 | # this needs to be false, otherwise hybrid mode doesn't work |
   | `tor.skip-proxy-for-clearnet-targets=true`  | # activate hybrid mode |

`CTRL-X` => `Yes` => `Enter` to save

RASPIBLITZ CONFIG FILE
`sudo nano /mnt/hdd/raspiblitz.conf` since Raspiblitz has some LND pre-check scripts which otherwise overwrite your settings.
   | Command | Description |
   | --- | --- |
   | `publicIP='207.154.241.101'`                | # add your VPS Public-IP |
   | `lndPort='9735'`                            | # define the LND port |
   | `lndAddress='207.154.241.101'`              | # define your LND public IP address |

`CTRL-X` => `Yes` => `Enter` to save

LND Systemd Startup adjustment

   | Command | Description |
   | --- | --- |
   | `sudo nano /etc/systemd/system/lnd.service` | edit the line 15 where it starts your LND binary, and add the following parameter: `ExecStart=/usr/local/bin/lnd ${lndExtraParameter}` |
   | `sudo systemctl restart lnd.service` | apply changes and restart your lnd.service. It will ask you to reload the systemd services, copy the command, and run it with sudo. This can take a while, depends how long your last restart was. Be patient. | 
   | `sudo tail -n 30 -f /mnt/hdd/lnd/logs/bitcoin/mainnet/lnd.log` | to check whether LND is restarting properly |
   | `lncli getinfo` | to validate that your node is now online with two uris, your pub-id@VPS-IP and pub-id@Tor-onion |

   ```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9735",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```        
 - [ ] Restart your LND Node with `sudo reboot`
</p>
</details>

<details><summary>Click here to expand Umbrel / Citadel settings</summary>
<p>
<!-- Add further comments for Umbrel and validate how to adjust starting LND docker with those changes, and making them persistent -->

   LND.conf adjustments, open with `sudo nano /home/umbrel/umbrel/lnd/lnd.conf`

   
[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735` | # to add your VPS Public-IP | 
   | `nat=false`                       | # deactivate NAT |

[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                          | # ensure Tor is active | 
   | `tor.v3=true`                              | # with the latest version. v2 is going to be deprecated this summer | 
   | `tor.streamisolation=false`                | # this needs to be false, otherwise hybrid mode doesn't work | 
   | `tor.skip-proxy-for-clearnet-targets=true` | # activate hybrid mode | 

`CTRL-X` => `Yes` => `Enter` to save

LND Restart to incorporate changes to `lnd.conf`
   | Command | Description |
   | --- | --- | 
   | `cd umbrel && docker-compose restart lnd` | this can take a while, depends how long your last restart was. Be patient. | 
   | `tail -n 30 -f ~/umbrel/lnd/logs/bitcoin/mainnet/lnd.log` | check whether LND is restarting properly | 
   | `~/umbrel/bin/lncli getinfo` | validate that your node is now online with two uris, your pub-id@VPS-IP and pub-id@Tor-onion | 
```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9735",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```
 - [ ] Restart your LND Node with `sudo reboot`

**Warning**: This guide did not verify yet, if and how the docker LND service on Umbrel & Citadel needs to be adjusted to channel clearnet packets via tunnel. We will add peer-reviewed adjustments here from Umbrel / Citadel devs. Until then, consider this highly experimental, it might fail.
</p>
</details>


### 9) LND Node: Start your VPN Client again
The reboot killed your tmux session running the OpenVPN client. Remember it from [section 6](#6-lnd-node-install-and-test-the-vpn-tunnel)? Here is how you restart it:
```
$ sudo apt-get install openvpn tmux
$ tmux new -s vps
$ sudo openvpn --config /home/admin/VPNcert/bringmesomesats.ovpn
```
`CTRL-B + CTRL-D`
Do the same for LND Node 2.

Additional hint: The author did not test the option to run the client automatically via systemctl, which obviously is better since it autostarts when you reboot your node. If you want to add this to be more independent from node starts, follow the guide to [add the autostart here](https://github.com/kylemanna/docker-openvpn/blob/master/docs/systemd.md). There are plenty of documentation on OpenVPN you can dig into, for instance giving your nodes same IPs each time. Scrapped this here for simplicity. 


### 10) Celebrate and wrapper
Now the moment of truth: once you tested the reboot, checked the LND log, and `lncli getinfo` shows you both the Tor and the VPS Clearnet IP as uris, you're done. LN gossip will soon populate your IP offering, and aggregator sites like Amboss or 1ml will pick it up. Time to celebrate 🍻 
or troubleshoot where things could have gone wrong. If the former: Congratulations - you made it!

Hope you enjoyed this article. Please do share feedback and suggestions for improvement.
If this guide was of any help, I'd appreciate if you share the article with others, give me a follow on Twitter [![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/HandsdownI.svg?style=social&label=Follow%20%40HandsdownI)](https://twitter.com/HandsdownI)
or even donating some sats below

[<img src=https://user-images.githubusercontent.com/35168804/156722937-59239884-5398-48ae-a4f2-1d6c53dfd0fb.png width="100" height="100">](https://pwbtc.duckdns.org/lnurlp/2)

or via my LN-Email hakuna@btcadresse.de. I'm also always grateful for incoming channels to my node: [HODLmeTight](https://amboss.space/node/037f66e84e38fc2787d578599dfe1fcb7b71f9de4fb1e453c5ab85c05f5ce8c2e3)


## Appendix & FAQ

#### I'm stuck and have no idea why it's not working. Who can help?
Please add an issue on Github with your question and provide as much detail as possible. Keep it safe though, no macaroon or user-ids!

#### Why DigitalOcean - can't we pick a VPS where we can pay with Lightning, and anonymously
Consider this guide a work-in-progress. I've picked DigitalOcean since I know what I'm doing there. Heard good things about [Luna Node](https://www.lunanode.com/), it's cheaper and you can pay with sats, so will test this out next. Also happy to add further alternatives, leave comments if you think these can accomplish the same results. Fee free to provide suggestions here.

#### Can I add even more nodes connecting to the tunnel? If so, how?
Yes, you need to handle your port-forwarding appropriately, since every node needs their unique LND listen port. Eg Node 1 has 9735, Node 2 9736 and so on. Docker runs need to be called with further `-p for publish-options`, IPtable rules and UFW needs to be adjusted. But once you got this guide internalised, the principle should be clear. Otherwise, let me know.
