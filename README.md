# * Nguyen Minh Phu Quang *
# MSSV:22110064
# Firewall_ATTACK_Lab
## Task Request
### Network Configuration

Given a network (for docker-compose, follow this link, in Network/firewall folder) that comprises 2 subnets, the whole network is configured by default:

- The router can route packets between the networks.
- There is a web server on each subnet (badsite, iweb) which can be accessed from computers on either subnet.
- Computers on one subnet can ping computers on the other network.
  
  - e.g. **`(on outsider) ping 172.16.10.100`** or **`(on inner1) ping 10.9.0.5`**
  
- Computers on a subnet can telnet into computers on the other subnet except the web servers, the router can be telneted from computers on either subnet.
  
  - e.g. **`(on outer) telnet 172.16.10.100`**
    - user: `exam`
    - password: `exam`
  
- Due to command line mode, the web servers can be accessed from all computers with curl:
  
  - e.g **`(on outsider) curl http://172.16.10.110`** or **`(on inner1) curl http://10.9.0.10`**

     ![image](https://github.com/user-attachments/assets/92288b4c-34bb-432c-a680-e94a025e3544)

- This is my file directory on VsCode:

 ![image](https://github.com/user-attachments/assets/963481a6-56cf-4933-bb3e-dc793f905d27)

- **Apache's Dockerfile:**
````dockerfile
FROM ubuntu:20.04
ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update  \
    && apt-get -y install  \
        apache2 \
        nano   \
        iproute2 \
    && a2enmod rewrite \
    && a2enmod ssl \
    && a2enmod cgi \
    && a2enmod headers


CMD service apache2 start && tail -f /dev/null
````
- **Image's Dockerfile:**
````dockerfile
FROM ubuntu:20.04
ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update  \
    && apt-get -y install  \
        binutils \
#	    netcat \
        curl   \
        iproute2  \
        iputils-ping \
        mtr-tiny \
        nano   \
        net-tools \
#        unzip \
        iptables \
        openbsd-inetd \
        telnet \ 
        telnetd \
        tcpdump \
#	scapy \
     && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash exam && \
	echo "root:exam" | chpasswd && \
	echo "exam:exam" | chpasswd && \
	usermod -aG sudo exam


CMD /bin/bash
````
- **Docker-Compose.yml:**
````python
version: "3"
  
services: 
    outsider:
        build: 
            context: ./image
        image: base-image
        container_name: outsider-10.9.0.5
        tty: true
        cap_add:
            - ALL
        sysctls:
            - net.ipv4.ip_forward=1
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.5
        command: bash -c "
                      ip route add 172.16.10.0/24 via 10.9.0.254 &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "
    inner:
        build: 
            context: ./image
        image: base-image
        container_name: inner-172.16.10.100
        tty: true
        cap_add:
            - ALL
        networks:
            net-172.16.10.0:
                ipv4_address: 172.16.10.100
        command: bash -c "
                      ip route del default &&
                      ip route add default via 172.16.10.10 &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "
    apache1:
        build: 
            context: ./apache
        image: apache-image
        container_name: iweb-172.16.10.110
        tty: true
        cap_add:
            - ALL
        networks:
            net-172.16.10.0:
                ipv4_address: 172.16.10.110
        command: bash -c "
                      ip route del default &&
                      ip route add default via 172.16.10.10 &&
                      service apache2 start &&
                      tail -f /dev/null
                "
    apache2:
        build: 
            context: ./apache
        image: apache-image
        container_name: badsite-10.9.0.10
        tty: true
        cap_add:
            - ALL
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.10
        command: bash -c "
                      ip route del default &&
                      ip route add 172.16.10.0/24 via 10.9.0.254 &&
                      service apache2 start &&
                      tail -f /dev/null
                "
    router:
        build:
            context: ./image
        image: router-image
        container_name: router
        tty: true
        cap_add:
            - ALL
        sysctls:
            - net.ipv4.ip_forward=1
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.254
            net-172.16.10.0:
                ipv4_address: 172.16.10.10
        command: bash -c "
                      ip route del default &&
                      ip route add default via 10.9.0.1 &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "

networks:
    net-10.9.0.0:
        name: net-10.9.0.0
        ipam:
            config:
                - subnet: 10.9.0.0/24
    net-172.16.10.0:
        name: net-172.16.10.0
        ipam:
            config:
                - subnet: 172.16.10.0/24
````
### Now i will test some request in the task overview to see if it can run properly:
- ***Overview of the task***

![image](https://github.com/user-attachments/assets/92ed61d5-5c93-4217-b2fb-5749820aa217)

- First i will setup the rules in my setup-iptables.sh file
````bash
#!/bin/bash

# Flush all existing rules
iptables -F
iptables -X

# Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -d 127.0.0.0/8 -j REJECT

# Allow ping (ICMP echo requests)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Allow established and related incoming connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Block all other incoming traffic to the router
iptables -A INPUT -j REJECT

# Prevent 10.9.0.0/24 from accessing the internal web server (iweb)
iptables -A FORWARD -s 10.9.0.0/24 -d 172.16.10.110 -j REJECT

# Prevent 172.16.10.0/24 from accessing the badsite
iptables -A FORWARD -s 172.16.10.0/24 -d 10.9.0.10 -j REJECT
mkdir -p /etc/iptables
# Save the iptables rules
iptables-save > /etc/iptables/rules.v4
````
- Then i setup router's Dockerfile and copy setup-iptables.sh into it:
````python

FROM ubuntu:20.04
ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get -y install iproute2 iptables openbsd-inetd telnet telnetd tcpdump nano && \
    rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash exam && \
    echo "root:exam" | chpasswd && \
    echo "exam:exam" | chpasswd && \
    usermod -aG sudo exam

COPY setup-iptables.sh router/setup-iptables.sh
RUN chmod +x router/setup-iptables.sh

CMD bash -c "router/setup-iptables.sh && /etc/init.d/openbsd-inetd start && tail -f /dev/null"
````
- Final step, i update my docker-compse.yml :
```python

  
services: 
    outsider:
        build: 
            context: ./image
        image: base-image
        container_name: outsider-10.9.0.5
        tty: true
        cap_add:
            - ALL
        sysctls:
            - net.ipv4.ip_forward=1
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.5
        command: bash -c "
                      ip route add 172.16.10.0/24 via 10.9.0.254 &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "
    inner:
        build: 
            context: ./image
        image: base-image
        container_name: inner-172.16.10.100
        tty: true
        cap_add:
            - ALL
        networks:
            net-172.16.10.0:
                ipv4_address: 172.16.10.100
        command: bash -c "
                      ip route del default &&
                      ip route add default via 172.16.10.10 &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "
    apache1:
        build: 
            context: ./apache
        image: apache-image
        container_name: iweb-172.16.10.110
        tty: true
        cap_add:
            - ALL
        networks:
            net-172.16.10.0:
                ipv4_address: 172.16.10.110
        command: bash -c "
                      ip route del default &&
                      ip route add default via 172.16.10.10 &&
                      service apache2 start &&
                      tail -f /dev/null
                "
    apache2:
        build: 
            context: ./apache
        image: apache-image
        container_name: badsite-10.9.0.10
        tty: true
        cap_add:
            - ALL
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.10
        command: bash -c "
                      ip route del default &&
                      ip route add 172.16.10.0/24 via 10.9.0.254 &&
                      service apache2 start &&
                      tail -f /dev/null
                "
    router:
        build:
            context: ./router
        image: router-image
        container_name: router
        tty: true
        cap_add:
            - ALL
        sysctls:
            - net.ipv4.ip_forward=1
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.254
            net-172.16.10.0:
                ipv4_address: 172.16.10.10
        command: bash -c "
                      ip route del default &&
                      ip route add default via 10.9.0.1 &&
                      router/setup-iptables.sh &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "

networks:
    net-10.9.0.0:
        name: net-10.9.0.0
        ipam:
            config:
                - subnet: 10.9.0.0/24
    net-172.16.10.0:
        name: net-172.16.10.0
        ipam:
            config:
                - subnet: 172.16.10.0/24
````
- Finally, we can build the docker up:

![image](https://github.com/user-attachments/assets/230c2afd-71b8-4c50-baf8-b801a0c145a3)

![image](https://github.com/user-attachments/assets/9c916e22-2667-4322-a564-892d1bca6e26)

- The result in Docker-Desktop:

![image](https://github.com/user-attachments/assets/a5a5c249-a90e-4df7-8d4b-01a83aec2435)

- Remember the setup-iptables.sh? There's a line:
````
# Save the iptables rules
iptables-save > /etc/iptables/rules.v4
````
- All the rules in "a, b, c" tasks is in this rules.v4 file, let see it:
````bash
# Generated by iptables-save v1.8.4 on Mon Jul 15 03:05:20 2024
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -d 127.0.0.0/8 -j REJECT --reject-with icmp-port-unreachable
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-port-unreachable
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -j DROP
-A FORWARD -s 10.9.0.0/24 -d 172.16.10.110/32 -j REJECT --reject-with icmp-port-unreachable
-A FORWARD -s 172.16.10.0/24 -d 10.9.0.10/32 -j REJECT --reject-with icmp-port-unreachable
-A FORWARD -s 10.9.0.0/24 -d 172.16.10.110/32 -p tcp -m tcp --dport 80 -j DROP
-A FORWARD -s 172.16.10.0/24 -d 10.9.0.10/32 -j DROP
-A FORWARD -s 10.9.0.0/24 -d 172.16.10.110/32 -p tcp -m tcp --dport 80 -j DROP
-A FORWARD -s 172.16.10.0/24 -d 10.9.0.10/32 -j DROP
COMMIT
# Completed on Mon Jul 15 03:05:20 2024
# Generated by iptables-save v1.8.4 on Mon Jul 15 03:05:20 2024
*nat
:PREROUTING ACCEPT [5:372]
:INPUT ACCEPT [3:252]
:OUTPUT ACCEPT [5:342]
:POSTROUTING ACCEPT [14:964]
:DOCKER_OUTPUT - [0:0]
:DOCKER_POSTROUTING - [0:0]
-A OUTPUT -d 127.0.0.11/32 -j DOCKER_OUTPUT
-A POSTROUTING -d 127.0.0.11/32 -j DOCKER_POSTROUTING
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p tcp -m tcp --dport 53 -j DNAT --to-destination 127.0.0.11:40815
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p udp -m udp --dport 53 -j DNAT --to-destination 127.0.0.11:46221
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p tcp -m tcp --sport 40815 -j SNAT --to-source :53
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p udp -m udp --sport 46221 -j SNAT --to-source :53
COMMIT
# Completed on Mon Jul 15 03:05:20 2024
````
- Explain:
   - INPUT chain: Allows loopback traffic and pings, drops/rejects all other traffic.
   - FORWARD chain: Blocks HTTP traffic from 10.9.0.0/24 to 172.16.10.110, and all traffic from 172.16.10.0/24 to 10.9.0.10.
   - OUTPUT chain: Default accept policy, with Docker-specific handling for DNS traffic.
   -+ NAT table: Handles Docker-specific NAT and DNS redirection rules.
- Now that we have set things up, let's get to do the task by testing:

#### Task a: Setup rules on router to block all access into it except ping

```
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
```
- This command ccepts ICMP echo requests (ping).

- Let see cotainers with "***docker ps***" :
![image](https://github.com/user-attachments/assets/8200dfea-3efd-4d56-8a94-bc02d6bd44a9)

- Now let's test (on outsider) ping 172.16.10.100:
 - Access the Outsider Container: Access the outsider container using the following command:
````
docker exec -it outsider-10.9.0.5 bash
````
 - Ping 172.16.10.100: Once inside the container, ping the IP address 172.16.10.100:
  ````
ping 172.16.10.100
````

![image](https://github.com/user-attachments/assets/a18aca07-6a59-4ad3-8cd4-fb955a5e4f67)

 - But we have to test if any access can do it, like curl:

![image](https://github.com/user-attachments/assets/b357a874-a700-4929-9356-da49294b702a)

(Can't connect)

#### Task b: Setup rules on router to prevent computers on subnet 10.9.0.0/24 from accessing the internal web server 
````
iptables -A FORWARD -s 10.9.0.0/24 -d 172.16.10.110 -j DROP
````
- Test this:
![image](https://github.com/user-attachments/assets/8b406dec-7527-4669-812a-24553eed3367)

### Task c: The badsite was found to contain malwares and source of delivering bots. Setup rules on router to stop computers on subnet 172.16.10.0/24 from accessing the badsite.

````
# Block access from 172.16.10.0/24 subnet to badsite (10.9.0.10)
-A FORWARD -s 172.16.10.0/24 -d 10.9.0.10 -j DROP
````
- Run this to see the iptables rule:
````
PS D:\Firewall_ATTACK_Lab> docker exec -it router bash
root@70a8e9b4292e:/# iptables -L -v
````
````
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  lo     any     anywhere             anywhere
    0     0 REJECT     all  --  any    any     anywhere             127.0.0.0/8          reject-with icmp-port-unreachable
    0     0 ACCEPT     icmp --  any    any     anywhere             anywhere             icmp echo-request
    5 11441 ACCEPT     all  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
    0     0 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-port-unreachable

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    7   420 DROP       all  --  any    any     10.9.0.0/24          iweb-172.16.10.110.net-172.16.10.0 
    0     0 DROP       tcp  --  any    any     10.9.0.0/24          inner-172.16.10.100.net-172.16.10.0  tcp dpt:telnet
   14   840 DROP       all  --  any    any     172.16.10.0/24       badsite-10.9.0.10.net-10.9.0.0 

Chain OUTPUT (policy ACCEPT 7 packets, 445 bytes)
 pkts bytes target     prot opt in     out     source               destination
````
- We have this line :
````
14   840 DROP       all  --  any    any     172.16.10.0/24       badsite-10.9.0.10.net-10.9.0.0
````
 - 840: The number of packets that have been dropped by this rule so far.
 - DROP: The action taken by this rule, which is to drop (discard) the packets.
 - all: This rule applies to all protocols (TCP, UDP, ICMP, etc.).
 - 172.16.10.0/24: The source IP address range. This means the rule applies to packets originating from any IP address within the subnet 172.16.10.0/24.
 - badsite-10.9.0.10.net-10.9.0.0: The destination IP address. This label represents the specific destination 10.9.0.10.

   - This line in my iptables configuration means that all packets originating from the subnet 172.16.10.0/24 and destined for 10.9.0.10 are being dropped. This effectively blocks computers on the subnet 
     172.16.10.0/24 from accessing the host at 10.9.0.10, which has been labeled as badsite.








- 
