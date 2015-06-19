# ec2-hostname



Public and private IPs

This means that for getting the public and private ips we only have to make two calls like this:
curl http://169.254.169.254/latest/meta-data/local-ipv4
and
curl http://169.254.169.254/latest/meta-data/public-ipv4

Hostnames

We need a way to identify from inside the instance what hostname this should be configured. There are probably several ways to do this, but the most common is to specify this when the instance is started using the —user-data option of the ec2-run-instances command (or the short form -d). This will pass the custom data and make it available to the instance.

You will probably want to customize this based on your needs. Myself I assumed that I will use the same domain for all instances and I need to pass only the hostname. Since I don’t need any other parameters to the machine I can just do this:

ec2-run-instances <AMI> -d "myhostname" ...other params...
If you use more user data, then you will probably use it like “hostname=myhostname <other_variables>”. (in this case you will need to update the script bellow like this: HOSTNAME=

echo $USER_DATA | cut -f 1 -d , | cut -f 2 -d =

Now making a http request like this will give us the hostname this instance should have:

curl http://169.254.169.254/latest/user-data

DNS Server configuration

Now that we have available inside the EC2 instance all the needed information, we need a way to allow the instance to update the DNS server. As I mentioned before, I will use for this a bind9 server, but this can by any other DNS server that allows this action to be scripted somehow (either using some api calls, or any way to use dynamic DNS update). For bind9 we will use the nsupdate utility to update the DNS server securely using the dnssec key mechanism.

Personally I don’t use the full domain for this, but delegate two subdomains (to the same nameservers) like this:


; ec2 zones:
ec2.domain.com.      NS      ns1.domain.com.
ec2.domain.com.      NS      ns2.domain.com.
ec2-int.domain.com.  NS      ns1.domain.com.
ec2-int.domain.com.  NS      ns2.domain.com.
and allow update access only to those zones. Of course if you prefer that you can give direct access to your full domain zone.

Generate a key using the dnssec-keygen utility like this:

dnssec-keygen -a HMAC-MD5 -b 512 -n USER user.domain.com.
and this will create two files like this:

Kuser.domain.com.+157+47950.key
Kuser.domain.com.+157+47950.private
Using the information from the public key add to your dns server configuration the key:

key user.domain.com. {
    algorithm HMAC-MD5;
    secret "xAw7F/axmVSxsZ+V4LAZnkeYObjOaJjbVKf21Zl4WhxtRHdlhqWSeCdd fIVR6MhC8LSQoim7NfkWD2j7WT5AHw==";
};
where secret is the value from the public key, that in my example looks like this:

cat Kuser.domain.com.+157+47950.key
user.domain.com. IN KEY 0 3 157 xAw7F/axmVSxsZ+V4LAZnkeYObjOaJjbVKf21Zl4WhxtRHdlhqWSeCdd fIVR6MhC8LSQoim7NfkWD2j7WT5AHw==
Finally we need to allow update access for the key:

zone "ec2.domain.com"
{
    type master;
    file "/etc/bind/zone/ec2.domain.com";
    allow-update { key user.domain.com.; };
    allow-query { any; };
};

zone "ec2-int.domain.com"
{
    type master;
    file "/etc/bind/zone/ec2-int.domain.com";
    allow-update { key user.domain.com.; };
    allow-query { any; };
};
Bind will need to be restarted after making these changes.

Using nsupdate to update the hostname

Next we will need to upload the key we created on the EC2 image (later we will save it inside the AMI once all runs well) and test to see if it is working properly.


cat<<EOF | /usr/bin/nsupdate -k Kuser.domain.com.+157+47950.private -v
server ns1.domain.com
zone ec2.domain.com
update delete test.ec2.domain.com A
update add test.ec2.domain.com 60 A <some_IP>
show
send
EOF
If this is working properly we can move on and put all this toghether in a script that will be running at the instance start time. If not, go back and see in your dns server logs if there are any issues why this is not working.

#!/bin/bash

#you will need to have the key available in the instance in the same dir as this script
DNS_KEY=Kuser.domain.com.+157+47950.private
DOMAIN=domain.com

USER_DATA=`/usr/bin/curl -s http://169.254.169.254/latest/user-data`
HOSTNAME=`echo $USER_DATA`
#set also the hostname to the running instance
hostname $HOSTNAME.$DOMAIN

PUBIP=`/usr/bin/curl -s http://169.254.169.254/latest/meta-data/public-ipv4`
cat<<EOF | /usr/bin/nsupdate -k $DNS_KEY -v
server ns1.$DOMAIN
zone ec2.$DOMAIN
update delete $HOSTNAME.ec2.$DOMAIN A
update add $HOSTNAME.ec2.$DOMAIN 60 A $PUBIP
send
EOF

LOCIP=`/usr/bin/curl -s http://169.254.169.254/latest/meta-data/local-ipv4`
cat<<EOF | /usr/bin/nsupdate -k $DNS_KEY -v
server ns1.$DOMAIN
zone ec2-int.$DOMAIN
update delete $HOSTNAME.ec2-int.$DOMAIN A
update add $HOSTNAME.ec2-int.$DOMAIN 60 A $LOCIP
send
EOF






















Mark it as executable

$ chmod o+x ec2-hostanme.sh



and add the following line to /etc/rc.local so that it runs every time the instance restart

/usr/local/ec2/ec2-hostname.sh



create a new instance, pass the desired hostname in the user-data option.

ec2-run-instances ami-99f510f1 --user-data "YOUR-HOSTNAME" \
  --instance-type t1.micro --group default --region us-east-1 \
  --key YOUR-KEY 
