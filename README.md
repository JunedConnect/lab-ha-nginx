# Deploying a Highly Available Nginx Web Server with SSL and Server Health Monitoring on AWS

<br>

## Overview

In this guide, you will set up a simple load balancing system using Nginx on Amazon EC2 instances. The setup includes:

- **Two Backend Servers**: These will serve simple HTML pages displaying which server is active.
- **Reverse Proxy / Load Balancer**: Distributes incoming traffic between the two backend servers.
- **Status Page**: Displays the status (online or offline) of the two backend servers.
- **SSL Encryption**: Secures your connection using Let's Encrypt SSL certificates.
- **Subdomains**: Configuring Cloudflare to route traffic to your reverse proxy and status pages.

By the end of this guide, you will have a load balancing system with secure HTTPS connections and a monitoring page to check the health of your backend servers.

<br>

## Step 1: Launch 4 EC2 Instances with Nginx

### 1.1 Create EC2 Instances

Log into your AWS console and navigate to EC2.

Create **four** EC2 instances:

- Two **Backend Servers** (These will run Nginx to serve simple HTML pages)
- One **Reverse Proxy / Load Balancer Server** (To balance the load between the backend servers)
- One **Status Page Server** (To monitor the status of the backend servers)

### 1.2 Configure Security Groups

- Allow **all traffic** for now to make setup easier. Once the project is complete, you should restrict access to only necessary ports for security.

### 1.3 Ensure Network Setup

Ensure **All EC2 Instances Are in the Same VPC and Subnet** for internal communication.

### 1.4 Connect to EC2 Instances

Once the instances are launched, connect to each EC2 instance using SSH.

---

## Step 2: Install Nginx on All EC2 Instances

### 2.1 Install Nginx

On each of the EC2 instances, run the following commands to install Nginx:

```bash
sudo yum update -y
sudo yum install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 2.2 Verify Installation

Now, Nginx should be installed and running on all EC2 instances.

<br>

## Step 3: Set Up Two Backend Servers

### 3.1 Purpose of Backend Servers

The backend servers will serve simple web pages, which will be used to verify that the load balancer is working correctly.

### 3.2 Configure Backend Server 1

On **Backend Server 1**, navigate to the Nginx HTML directory:

```bash
cd /usr/share/nginx/html
```

Replace `index.html` with:

```html
<html>
<body>
    <h1>Backend Server 1</h1>
    <p>This is server 1.</p>
</body>
</html>
```

### 3.3 Configure Backend Server 2

On **Backend Server 2**, navigate to the same directory and replace `index.html` with:

```html
<html>
<body>
    <h1>Backend Server 2</h1>
    <p>This is server 2.</p>
</body>
</html>
```

### 3.4 Test Backend Servers

Ensure each backend server is running by visiting its respective public IP address in a browser.

<br>

## Step 4: Set Up Reverse Proxy / Load Balancer

### 4.1 Find Private IP Addresses

Find the **Private IP addresses** of Backend Server 1 and Backend Server 2 in the AWS EC2 instance details.

### 4.2 Configure Nginx on Reverse Proxy

Edit the Nginx config file:

```bash
sudo nano /etc/nginx/nginx.conf
```

### 4.3 Update Nginx Configuration

Insert the following inside the `http {}` block:

```nginx
upstream backendserver {
    server <PRIVATE IP of Backend Server 1>;
    server <PRIVATE IP of Backend Server 2>;
}

server {
    listen 80;
    server_name app.<YOUR DOMAIN>; # Replace with your actual domain name
    
    location / {
        proxy_pass http://backendserver/;
    }
}
```

### 4.4 Restart Nginx and Test

Restart Nginx:

```bash
sudo systemctl restart nginx
```

Test the load balancer by visiting the public IP of the Reverse Proxy server.

---

## Step 5: Set Up Status Page

### 5.1 Install Cron and Create Script

```bash
sudo yum install cronie -y
sudo systemctl enable crond
sudo systemctl start crond
```

Create the status script:

```bash
sudo nano /usr/local/bin/status.sh
```

Paste the script:

```bash
#!/bin/bash
BS1IP=<PRIVATE IP of Backend Server 1>
BS2IP=<PRIVATE IP of Backend Server 2>

while true; do
    if ping -w 1 $BS1IP &> /dev/null; then
        BS1Status="Online"
    else
        BS1Status="Offline"
    fi

    if ping -w 1 $BS2IP &> /dev/null; then
        BS2Status="Online"
    else
        BS2Status="Offline"
    fi

    cat << EOF > /usr/share/nginx/html/index.html
    <html>
    <body>
        <h1>Backend Server Status</h1>
        <p>Server 1: $BS1Status</p>
        <p>Server 2: $BS2Status</p>
    </body>
    </html>
EOF
done
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/status.sh
```

Set up a cron job:

```bash
sudo crontab -e
```

Add:

```bash
* * * * * /usr/local/bin/status.sh
```

<br>

## Step 6: Set Up Subdomains with Cloudflare

### 6.1 Configure Cloudflare DNS

1. Log in to **Cloudflare** and add your domain.
2. Navigate to **DNS settings**.
3. Create:
   - **A Record** for `app.<YOUR DOMAIN>` → Public IP of Reverse Proxy.
   - **A Record** for `status.<YOUR DOMAIN>` → Public IP of Status Page.

### 6.2 Ensure Cloudflare Does Not Interfere

Make sure Cloudflare is not interfering too much, as it could mess up the SSL encryption step. You may need to disable Cloudflare’s proxying (orange cloud icon) to avoid issues.

Once propagated, Cloudflare will resolve your subdomain requests correctly.

<br>

## Step 7: Set Up SSL with Let's Encrypt

### 7.1 Install Certbot

```bash
sudo yum install -y certbot python2-certbot-nginx
```

### 7.2 Obtain SSL Certificates

```bash
sudo certbot --nginx -d app.<YOUR DOMAIN> -d status.<YOUR DOMAIN>
```

### 7.3 Verify SSL Configuration

Ensure that Let's Encrypt has correctly modified the Nginx configuration for HTTPS.

Restart Nginx:

```bash
sudo systemctl restart nginx
```

---

### Done!

Your Nginx Load Balancer with SSL and a Status Page is now live!

<br>

|Here is what it will look like (mine will have a different layout as I had customised the html files in Step 3 & 6):|
|-------|
| ![WebServer1](https://raw.githubusercontent.com/JunedConnect/Lab_HA-Nginx/main/images/WebServer1.png) |
| ![WebServer1](https://raw.githubusercontent.com/JunedConnect/Lab_HA-Nginx/main/images/WebServer2.png) |
| ![WebServer1](https://raw.githubusercontent.com/JunedConnect/Lab_HA-Nginx/main/images/WebServerCertificate.png) |
| ![WebServer1](https://raw.githubusercontent.com/JunedConnect/Lab_HA-Nginx/main/images/WebServerStatus.png) |
