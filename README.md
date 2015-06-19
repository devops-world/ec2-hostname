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






Mark it as executable

$ chmod o+x ec2-hostanme.sh



and add the following line to /etc/rc.local so that it runs every time the instance restart

/usr/local/ec2/ec2-hostname.sh



create a new instance, pass the desired hostname in the user-data option.

ec2-run-instances ami-99f510f1 --user-data "YOUR-HOSTNAME" \
  --instance-type t1.micro --group default --region us-east-1 \
  --key YOUR-KEY 
