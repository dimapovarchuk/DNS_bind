# DNS_bind_ubuntu_server
On both DNS servers, ns1 and ns2, update apt: 

sudo apt-get update 

Now install BIND: 

sudo apt-get install bind9 bind9-doc -y 


<img width="470" alt="image" src="https://user-images.githubusercontent.com/52627259/222191045-c23caad4-f800-4d88-848f-6637ca2077f1.png">


Before continuing, let’s set BIND to IPv4 mode. On both servers, edit the bind9 service parameters file: 

sudo nano /etc/default/bind9 

Add “-4” to the OPTIONS variable. It should look like the following: 

OPTIONS="-4 -u bind" 

 <img width="472" alt="image" src="https://user-images.githubusercontent.com/52627259/222191264-a0623dab-3ac7-4ea8-bc19-0705149f9b0b.png">


Save and exit. 

One of the important configuration file for bind is “/etc/bind/named.conf.options“, from this file we can set the followings parameters: 

Allow Query to your dns from your private network (As the name suggests only the systems from your private network can query dns sever for name to ip translation and vice-versa) 

Allow recursive query 

Specify the DNS port ( 53) 

Forwarders (DNS query will be forwarded to the forwarders when your local DNS server is unable to resolve query) 

On ns2, edit the named.conf.options file: 

sudo nano /etc/bind/named.conf.options 

acl "trusted" { 

        10.128.10.11;   # ns1 

        10.128.20.12;   # ns2 - can be set to localhost 

        10.128.100.101;  # host1 

        10.128.200.102;  # host2 

}; 

options { 

        directory "/var/cache/bind"; 

         

        recursion yes;                 # enables resursive queries 

        allow-recursion { trusted; };  # allows recursive queries from "trusted" clients 

        listen-on { 10.128.10.11; };   # ns1 private IP address - listen on private network only 

        allow-transfer { none; };      # disable zone transfers by default 

 

        forwarders { 

                8.8.8.8; 

                8.8.4.4; 

        }; 

... 

}; 

 <img width="371" alt="image" src="https://user-images.githubusercontent.com/52627259/222191655-c6bd2f6e-578f-44c2-8f19-3da5602f7618.png">


 <img width="341" alt="image" src="https://user-images.githubusercontent.com/52627259/222191718-fb2dfa92-d6ea-43fc-b7df-193b1d2fe66e.png">


 

Next Important Configuration file is “/etc/bind/named.conf.local“, in this file we will define the zone files for our domain, edit the file add the following entries: 

sudo nano /etc/bind/named.conf.local 

zone "nyc3.example.com" { 

    type master; 

    file "/etc/bind/zones/db.nyc3.example.com"; # zone file path 

    allow-transfer { 10.128.20.12; };         # ns2 private IP address - secondary 

}; 

zone "128.10.in-addr.arpa" { 

    type master; 

    file "/etc/bind/zones/db.10.128";  # 10.128.0.0/16 subnet 

    allow-transfer { 10.128.20.12; };  # ns2 private IP address - secondary 

}; 

 <img width="445" alt="image" src="https://user-images.githubusercontent.com/52627259/222191894-6efbb3c6-7fcc-4e89-8d1a-d6e94be8f25e.png">


Now save and exit named.conf.local. 

Create Forward Zone File 

mkdir -p /etc/bind/zones/ 

# Copy default forward zone 

sudo cp /etc/bind/db.local /etc/bind/zones/forward.anpavlovsk.com 

# Copy default reverse zone 

sudo cp /etc/bind/db.127 /etc/bind/zones/reverse.anpavlovsk.com 

# List contents of the /etc/bind/zones/ directory 

ls /etc/bind/zones/ 

 <img width="473" alt="image" src="https://user-images.githubusercontent.com/52627259/222192075-bff313bc-198a-4bb9-9e25-77a2e3ff2733.png">


cd /etc/bind/zones 

sudo cp ../db.local ./db.nyc3.example.com 

sudo nano /etc/bind/zones/db.nyc3.example.com 

Our final example forward zone file looks like the following: 

$TTL    604800 

@       IN      SOA     ns1.nyc3.example.com. admin.nyc3.example.com. ( 

      3		; Serial 

 604800		; Refresh 

  86400		; Retry 

2419200		; Expire 

 604800 )	; Negative Cache TTL 

; 

; name servers - NS records 

     IN      NS      ns1.nyc3.example.com. 

     IN      NS      ns2.nyc3.example.com. 

 

; name servers - A records 

ns1.nyc3.example.com.          IN      A       10.128.10.11 

ns2.nyc3.example.com.          IN      A       10.128.20.12 

 

; 10.128.0.0/16 - A records 

host1.nyc3.example.com.        IN      A      10.128.100.101 

host2.nyc3.example.com.        IN      A      10.128.200.102 

 <img width="459" alt="image" src="https://user-images.githubusercontent.com/52627259/222192288-9f8e196b-cecd-4e6f-8196-06d24ce2f3fa.png">


Create Reverse Zone File(s) 

cd /etc/bind/zones 

sudo cp ../db.127 ./db.10.128 

sudo nano /etc/bind/zones/db.10.128 

The last example of a return zone file looks like this: 

$TTL    604800 

@       IN      SOA     nyc3.example.com. admin.nyc3.example.com. ( 

                              3         ; Serial 

                         604800         ; Refresh 

                          86400         ; Retry 

                        2419200         ; Expire 

                         604800 )       ; Negative Cache TTL 

; name servers 

      IN      NS      ns1.nyc3.example.com. 

      IN      NS      ns2.nyc3.example.com. 

 

; PTR Records 

11.10   IN      PTR     ns1.nyc3.example.com.    ; 10.128.10.11 

12.20   IN      PTR     ns2.nyc3.example.com.    ; 10.128.20.12 

101.100 IN      PTR     host1.nyc3.example.com.  ; 10.128.100.101 

102.200 IN      PTR     host2.nyc3.example.com.  ; 10.128.200.102 

 <img width="359" alt="image" src="https://user-images.githubusercontent.com/52627259/222192503-975d56bc-6108-4264-9530-fa73b3f93f6e.png">


Run the following command to check the validity of your configuration files: 

sudo named-checkconf 

 <img width="419" alt="image" src="https://user-images.githubusercontent.com/52627259/222192617-b3f08932-d922-4ae7-84c6-84fcb4b23ab0.png">


sudo named-checkzone nyc3.example.com db.nyc3.example.com 

sudo named-checkzone 128.10.in-addr.arpa /etc/bind/zones/db.10.128 

 <img width="464" alt="image" src="https://user-images.githubusercontent.com/52627259/222192767-61d8ece6-fd8f-45eb-b66b-d7e687beac1f.png">


Once that checks out, restart bind 

 

sudo service bind9 restart 

 <img width="461" alt="image" src="https://user-images.githubusercontent.com/52627259/222193016-bb4c1941-69ba-48e1-ab4d-313db10cbcdb.png">


 

On ns2, edit the named.conf.options file: 

sudo nano /etc/bind/named.conf.options 

acl "trusted" { 

        10.128.10.11;   # ns1 

        10.128.20.12;   # ns2 - can be set to localhost 

        10.128.100.101;  # host1 

        10.128.200.102;  # host2 

}; 

        recursion yes; 

        allow-recursion { trusted; }; 

        listen-on { 10.128.20.12; };      # ns2 private IP address 

        allow-transfer { none; };          # disable zone transfers by default 

 

        forwarders { 

                8.8.8.8; 

                8.8.4.4; 

        }; 

 <img width="294" alt="image" src="https://user-images.githubusercontent.com/52627259/222193228-0c99e771-862f-43d9-8851-13143fe45065.png">


sudo nano /etc/bind/named.conf.local 

zone "nyc3.example.com" { 

    type slave; 

    file "slaves/db.nyc3.example.com"; 

    masters { 10.128.10.11; };  # ns1 private IP 

}; 

 

zone "128.10.in-addr.arpa" { 

    type slave; 

    file "slaves/db.10.128"; 

    masters { 10.128.10.11; };  # ns1 private IP 

}; 

 <img width="458" alt="image" src="https://user-images.githubusercontent.com/52627259/222193484-b1e89d18-dd6f-42b5-bd41-af627071a423.png">


sudo named-checkconf 

 <img width="457" alt="image" src="https://user-images.githubusercontent.com/52627259/222193605-295290b5-e3dd-408d-9183-1852d75f7188.png">


sudo service bind9 restart 

sudo systemctl status bind9 

 <img width="473" alt="image" src="https://user-images.githubusercontent.com/52627259/222193825-da5e193e-ce05-4f5d-8327-cd7442cfb6b3.png">


 

 

 
DNS_bind
