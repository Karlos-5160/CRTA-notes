### External Lab Setup Overview
    • We will install 2 role-assigned machines in external network :
    • Kali Linux [EX- 192.168.50.2]
    Web-Server [EX - 192.168.50.3, INT- 10.10.10.5]
    • The Web-Server machine has 2 networks and should be directly accessible from the
    attacker network.
    • However, the Employee-Machine is in the Internal Network.

- Network Adapter #3 ---> 192.168.50.0/24 
- Network Adapter #4 ---> 10.10.10.0/24

### Setting up Virtual Machines
    
## A. Attacker Machine Setup (Kali)
1. we will add a new host only adapter (#3) with dhcp off to kali machine
2. edit the /etc/network/interfaces file with the below static ip's
    • Installing and configuring
    • Gateway [192.168.50.1]
    • Attacker-Machine eth0 ---> [192.168.50.2/24] #3
    • Subnet mask ---> [255.255.255.0]
     

## B. Web-Server Setup (Metasploitable)
1. we will add two new host only adapter (#3 and #4 )with dhcp off to the web server
2. edit the /etc/network/interfaces file with the below static ip's
    • Installing and configuring
    • Gateway [192.168.50.1]
    • Web-Server eth0 ---> [192.168.50.3/24] #3
    • Web-Server eth1 ---> [10.10.10.5/24] #4
   