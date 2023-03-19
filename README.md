# Deploying a Front & Backend web app using AWS S3 and EC2 with Nginx and SSL/HTTPS

Deploying a Book Collection System developed on the MERN Stack (MongoDB, Express, React & Nodejs) on AWS S3 and EC2 with Nginx and SSL/HTTPS.

### Prerequisites:
- An AWS account
- A MongoDB Atlas account
- Node and npm installed


### Build Application

Open the _mern-app-client_ directory in demo project and run the following command in terminal to install dependencies: 

    npm install 

Build a _build_ folder (this will contain the build files that will load the React app on the server):

    npm run build

Create a database in MongoDB Atlas and create a user & password for it, select Connect and Connect your application. Copy the connection string and paste it in _config/default.json_.

## Create S3 Buckets for frontend

Access S3 via AWS console and create one S3 bucket name it with the desired domain name e.g. `abcd.com` and another S3 bucket with desired subdomain name e.g. `www.abcd.com`

Change Permissions and edit Bucket Policy on both Buckets to make them publicly accessible.

Under Properties, enable Static Website Hosting for the `abcd.com` bucket, select _Use this bucket to host a website_ and type 'index.html' for _index document_.
For `www.abcd.com` bucket select _Redirect requests_ and enter `abcd.com` for _Target bucket or domain_.

Upload all files from the _build_ folder to both buckets. Use endpoint url from the Static Hosting tab to access the website.

### Setup DNS service with Route53

Route 53 is a highly available and scalable Domain Name System service. The DNS Management is in charge of routing all domain traffic to a specified server or available service. Get a domain with Route53 or migrate from another domain service
(e.g. GoDaddy or NameCheap).

From AWS console, access Route53, select Hosted Zones and click _Create hosted zone_. Enter domain name `abcd.com` for Domain Name, Public Hosted Zone for Type and select Create.

### Create CloudFront Distribution

CloudFront is a Content Delivery Network (CDN) that offers the best possible performance when delivering data to the user by routing said user to an edge location that provides low latency. 
From AWS console, access CloudFront and create a Distribution. In _Origin Domain_ select S3 URL (`abcd.com`) and select Redirect HTTP to HTTPS under _Default Cache Behavior Settings_. Leave other settings as is.

### Create SSL Certificate

From AWS console, access Certificate Manager and select _Request a Certificate_ then _Request a public certificate_. Add domain name and subdomain name (with _Add another name to this certificate_). Click Next and select DNS validation, leave the rest as is and select Review. On the next page expand the certificate tab (may pop up as _View Certificate_) and click _Create record in Route53_ for all certificates (to add CNAME record into Route53 hosted zone)
then wait for the domain name to get validated. Return to CloudFront and select previously created distribution, select _Custom SSL Certificate_ and choose the newly created certificate.

To check if everything is working so far, copy the CloudFront Domain name value with https and paste in browser to access the static/frontend files on the S3 bucket.

### Connect Domain to CloudFront with Route53

Go to Route53, select the Hosted Zone and select _Create Record Set_. Then input _Name:_ www, _Type:_ A-IPv4 address, _Alias:_ Yes, select distribution in _Alias Target_ and click Create.

## Provision and connect to an EC2 instance for backend

Access EC2 from the AWS console and select _Launch Instance_. Select AMI (Amazon Linux AMI in this demo), Instance type, add Storage, create or use a Security Group with the following Inbound Traffic rules:
- Type: SSH, Protocol: TCP & Port: 22 (to connect to Instance)
- Type: HTTP, Protocol: TCP & Port: 80 (to allow http requests from anywhere)
- Type: HTTPS, Protocol: TCP & Port: 443 (to allow https)

Create or add existing key pair pem file and click Launch Instance.

Create an Elastic IP, so the private IP address will not reset on instance restart or shutdown, from EC2 dashboard select Elastic IPs under Network & Security.
Select _Allocate Elastic IP address_, _Amazon’s pool of IPv4 addresses_ and click Allocate. Then, select the row containing the Public IPv4 address and Allocation ID and select _Associate Elastic IP address_ via _Actions_ button. Choose Instance Resource type and select the instance and click Associate.

Change the permissions on the pem file in terminal:

    chmod 400 <path to key pair file>/<name of key pair file>.pem

SSH into the EC2 instance with this command:

    ssh -i <path to key pair file>/<name of key pair file>.pem ec2-user@<instance Public IP address>

### Setup EC2 Instance for backend

Connected to instance, run these commands to update packages and install Node.js & npm:

    sudo yum update -y
    
    curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
    
    sudo yum install nodejs -y

Verify installations:

    nodejs -v
    npm -v

Install Nginx:

    sudo yum install nginx -y

Set up Firewall rules to allow SSH and HTTP/HTTPS connections:

    sudo ufw allow OpenSSH

    sudo ufw allow 'Nginx Full'

    sudo ufw --force enable

### Deploy backend files on Instance

Clone repository on instance, cd into the _mern-app_ directory  and install dependencies:

    sudo npm install

### Test-configure Nginx to direct API calls to backend

cd to /etc/nginx/ and delete the default Nginx config file with command:

    sudo rm /etc/nginx/sites-enabled/default

Then, create and open a new Nginx config file:

    sudo nano /etc/nginx/sites-enabled/default

Copy the contents of txt/nginx-config-1.txt and paste into the config file.

Enter 'y' to save. Verify config change and then restart Nginx to implement change:

    sudo nginx -t

    sudo systemctl restart nginx

### Use Certbot to enable HTTPS on EC2 instance

Certbot is a free, open source software tool for automatically using Let’s Encrypt certificates on manually-administrated websites to enable HTTPS.  
Certbot uses dns_route53 plugin that automates the process of completing a dns-01 challenge (DNS01) by creating, and subsequently removing, TXT records using the Amazon Web Services Route 53 API.
So a Policy needs to be created for the dns_route53 plugin to connect to AWS.

Access Route53 via AWS console and copy the hosted zone ID.

Access IAM via AWS console, select _Policy_ and _Create Policy_. Copy this sample policy 
from the certbot GitHub page and paste it in the _Create Policy_ page: https://github.com/certbot/certbot/blob/master/certbot-dns-route53/examples/sample-aws-policy.json  
This policy creates route53:ListHostedZones, route53:GetChange and route53:ChangeResourceRecordSets permissions for Certbot.

Paste hosted zone ID in `YOURHOSTEDZONEID` line, click Review and Create Policy.

Still in IAM, select _Add User_, create a User, select _Programmatic Access_ for AWS access type, click Next,
select `Attach Existing Policy` then search and select previously created policy. Review and then Add User. Copy _Access
key ID_ and _Secret access key_.

### Install AWS cli on EC2 instance for Certbot

Install AWS cli and run it:

    sudo yum install awscli -y

    aws configure

Add the _Access key_, _Secret access key_, region and output format (json).

Run command and copy the output (hosted zone):

    aws route53 list-hosted-zones — output text

Install Certbot, Route53 plugin and install certificate:

    sudo yum install certbot -y

    sudo yum install python3-certbot-dns-route53 -y

    sudo certbot certonly --dns-route53 -d api.<Hosted zone name>

After installations, verify certificate:

    sudo certbot certificates

### Configure Nginx to accept HTTPS requests

cd to /etc/nginx/ and delete the default Nginx config file again:

    sudo rm /etc/nginx/sites-enabled/default

Then, create and open a new Nginx config file:

    sudo nano /etc/nginx/sites-enabled/default

Copy and paste the contents of txt/nginx-config-2.txt into the config file (replace DOMAIN.NAME with the domain name `abcd.com`)

Enter 'y' to save. Verify the config change and restart Nginx:

    sudo nginx -t

    sudo systemctl restart nginx

### Configure Route53

Head back over to Amazon Certificate Manager via AWS console and create a new certificate for api.`abcd.com`
and select _Create record in Route53_ to add CNAME records to Route53.

The DNS service will have to be configured in order to route all requests from api.`abcd.com` to EC2 instance.  
Access Route53 and select the previously hosted zone and create a Record Set with the following details:
- Name: api.abcd.com
- Type: IPv4 Address
- Value: EC2 Public IP or previously created Elastic Public IP

### Connect frontend and backend

In _mern-app-client_ folder, create a file called `.env.prod`, and paste this variable into the file: `REACT_APP_API_URL=http://api.abcd.com/api`

Install env-cmd:

    npm install env-cmd --save

Add a new script to package.json and save:

```
{
  "scripts": {
    "build:prod": "env-cmd -f .env.prod npm run build"
  }
}
```

In terminal, run `npm run build:prod`. Copy the build files and upload them to both S3 buckets after emptying both buckets.  
The frontend (client-side) has been updated and connected to the backend (server-side).
