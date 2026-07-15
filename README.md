# platform-engineering-lab-01

## Create the Application User
- Create a dedicated application user to run the application. This will improve security by running application as non-root user.
```bash
sudo adduser app
```
## Clone Source Code
- Clone the repository to your local machine.
```bash
git clone https://github.com/khantnaingset-kns/ape-aws-ec2-assessment-1.git
```

## Move the project to specific path
- Move the project to /usr/local/bin according to best practice.
```bash
mv ape-aws-ec2-assessment-1 /usr/local/bin/
```

## Install OS Dependencies
- Install required package before running application.
```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git nginx
```

## Create Application Log Directory
- Create Application Log Directory and log file and Make Sure app user have the enough permission.
```bash
sudo mkdir -p /var/log/storage-breaker
sudo chown -R app:app /var/log/storage-breaker
```

## Install Project Level Dependencies
- Navigate to project directory.
```bash
cd /usr/local/bin/ape-aws-ec2-assessment-1
```
- Activate Python Virtual Environment.
```bash
python3 -m venv .venv
source .venv/bin/activate
```
- Install project level dependencies with the commands which is provided by developer.
```bash
pip install --upgrade pip
pip install -r requirements.txt
```
## Run the Application
- Run the Application with port 3000 and other specific flags.
```bash
.venv/bin/uvicorn app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```

## Local Test
- Make Local Testing that the service is running well via health route.
```bash
curl -i http://localhost:3000/health
```

## Integrate with Nginx as Reverse_Proxy Server
- After local testing succeed, service should be exposed to public (internet) via Nginx.

- Delete default configuration under /etc/nginx/sites-available and /etc/nginx/sites-enabled/ in order not to handle requests via default site.
```bash
sudo rm -rf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
```

- Create new Nginx configuration file and configure as Reverse_Proxy.
```bash
cd /etc/nginx/sites-available
sudo nano app
```

```bash
server {
    listen 80;
    server_name app;

    location / {
        proxy_pass http://localhost:3000;
    }
}
```
- Use symbolic link to sites-enabled
```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app
```
- Check Nginx configuration is valid.
```bash
sudo nginx -t
sudo nginx -T
```

- Restart Nginx Service
```bash
sudo systemctl restart nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

- Test the service is exposed to public
```bash
curl -i http://<instance's public ip>/health
```

## Setup service as systemd service

- Setup serice as systemd service and make sure POST every boots up.
```bash
cd /etc/systemd/system/
sudo nano app.service
```
```bash
[Unit]
Description=App
After=network.target

[Service]
User=app
Group=app
WorkingDirectory=/usr/local/bin/ape-aws-ec2-assessment-1
ExecStart=/usr/local/bin/ape-aws-ec2-assessment-1/.venv/bin/uvicorn app:app --host 127.0.0.1 --port 3000 --workers 1 --no-access-log
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target"
```
- Restart and Enable Service
```bash
sudo systemctl restart app.service
sudo systemctl enable --now app.service
```

## APPLICATION IS UNHEALTHY

- After running application over 2hr, testing result is "unhealthy".
- Trace the Issue
```bash
df -h
```
- Now system storage is totally full showing 100% usage at root directory and hunt which file is cosuming the storage.
```bash
sudo du -sh *
sudo du -sh /var/*
sudo du -sh /var/log/*
sudo du -sh /var/log/storage-breaker/*
```

### Temporary Solution (Truncate)

- Reduce the application log file size. In this case, can't remove log file, if remove, the application will crash.
```bash
truncate -s 100M /var/log/storage-breaker/application.log
```

### Permanent Solution (Logrotate)
- Use logrotate tool (native linux tool)

```bash
cd /etc/logrotate.d
sudo nano app
```
```bash
/var/log/storage-breaker/application.log {
size 100M
rotate 10
compress
delaycompress
notifempty
missingok
create 0640 app app
}
```
- Log rotate will trigger when logfile size is over 100M. If size is over 100M, seperate new log file like application.log.1 and then compress with gz.
- Will only have 10 max log file.

- Test logrotate configuration work well.
```bash
test force rotate -> sudo logrotate -v -f /etc/logrotate.d/app
```

- Since Application is Runnig as systemd service, systemd will handle logrotate as .service and .timer.
```bash
cd /etc/systemd/system
sudo nano applog.service
```
```bash
[Unit]
Description=Rotate log files

[Service]
Type=oneshot
ExecStart=/usr/sbin/logrotate /etc/logrotate.conf
```
```bash
sudo nano applog.timer
```
```bash
[Unit]
Description=RunLogrotate

[Timer]
OnBootSec=30sec
OnUnitActiveSec=1min
Persistent=true
Unit=logrotate.service

[Install]
WantedBy=timers.target
```
```bash
sudo systemctl status applog.service
sudo systemctl status applog.timer
sudo systemctl enable --now applog.timer
```
