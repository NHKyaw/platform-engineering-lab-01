# platform-engineering-lab-01

##Create the Application User
Create a dedicated application user to run the application. This will improve security by running application as non-root user.
```bash
sudo adduser app
```
##Clone Source Code
Clone the repository to your local machine.
```bash
git clone https://github.com/khantnaingset-kns/ape-aws-ec2-assessment-1.git
```

##Move the project to specific path
Move the project to /usr/local/bin according to best practice.
```bash
mv ape-aws-ec2-assessment-1 /usr/local/bin/
```

##Install OS Dependencies
Install required package before running application.
```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git nginx
```

##Create Application Log Directory
Create Application Log Directory and log file and Make Sure app user have the enough permission.
```bash
sudo mkdir -p /var/log/storage-breaker
sudo chown -R app:app /var/log/storage-breaker
```

##Install Project Level Dependencies
Navigate to project directory.
```bash
cd /usr/local/bin/ape-aws-ec2-assessment-1
```
Activate Python Virtual Environment.
```bash
python3 -m venv .venv
source .venv/bin/activate
```
Install project level dependencies with the commands which is provided by developer.
```bash
pip install --upgrade pip
pip install -r requirements.txt
```
##Run the Application
Run the Application with port 3000 and other specific flags.
```bash
.venv/bin/uvicorn app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```

##Local Test
Make Local Testing that the service is running well via health route.
```bash
curl -i http://localhost:3000/health
```

##Integrate with Nginx as Reverse_Proxy Server
After local testing succeed, service should be exposed to public (internet) via Nginx.

Delete default configuration under /etc/nginx/sites-available and /etc/nginx/sites-enabled/ in order not to handle requests via default site.
```bash
sudo rm -rf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
```

Create new Nginx configuration file and configure as Reverse_Proxy.
- cd /etc/nginx/sites-available
- sudo nano app

```bash
server {
    listen 80;
    server_name app;

    location / {
        proxy_pass http://localhost:3000;
    }
}
```
Check Nginx configuration is valid.
- sudo nginx -t
- sudo nginx -T

Restart Nginx Service
- sudo systemctl restart nginx
- sudo systemctl enable --now nginx

