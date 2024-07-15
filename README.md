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

- Apache's Dokerfile:
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


