# Deploying to an ubuntu server 
Though out this process i will be using a linux server with the username of `bee`


### Upgrade system
run `sudo apt update` and `sudo apt update` 

### install required packages

```
sudo apt install python3-venv
```

```
sudo apt install libpq-dev
```

```
sudo apt install nginx
```

```
sudo apt install nginx 
```

```bash
sudo apt install curl
```

### Create a new virtual environment 
ensure you are in the root directory of your linux server i.e `/home/<username>/`

```bash
python3 -m venv venv
```

### Activate your virtual environment
```bash
source venv/bin/activate
```
(venv) should now be prepended to your path, this means the virtual environment was activated successfully

### Clone the github repo
```
git clone https://github.com/Codebee50/NNRA-circle.git 
```

### Upgrade pip
```
pip install --upgrade pip
```

### Install requirements
Navigate into the folder containing requirements.txt file
```bash
cd NNRA-circle
```

```
pip install -r requirements.txt
```

If postgresql is your database choice, install psycog2-binary, this is used to communicate with a postgreql database
```
pip install psycopg2-binary
```




## Configure environment variables 
For security purposes during deployment, This project is configured to fetch secret keys from a .env file 

**CREATE A .env file**

```bash
touch .env
```

**open created file**
```
sudo nano .env
```

The current configuration for .env files are 

```
EMAIL_HOST_USER_DEBUG=<>
EMAIL_HOST_PASSWORD_DEBUG=<>

AWS_ACCESS_KEY_ID=<>
AWS_SECRET_ACCESS_KEY=<>
AWS_STORAGE_BUCKET_NAME=<>

RENDER_DB_EXTERNAL_URL=<>
```
To avoid pushing secrets to github, i have removed all the secret values from this file and replaced them with `<>`

EMAIL_HOST_USER_DEBUG and EMAIL_HOST_PASSWORD_DEBUG are used to configure email sending credentials 

AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_STORAGE_BUCKET_NAME are used to configure an aws s3 bucket 

RENDER_DB_EXTERNAL_URL is the external url of my database created at https://render.com

`When using .env variables ensure there are no spaces between file names and file values e.g`
```
AWS_ACCESS_KEY_ID=uejehrreje9303=ssl\ssjrjr
```
`Notice that there are no spaces and the value uejehrreje9303=ssl\ssjrjr is not wrapped in quotes despite the fact that it contains numbers, strings and letters`



## Set up database
The current project setup is confugured to use a postgresql database provided by render you can create a new posgresql database on https://render.com/ and set RENDER_DB_EXTERNAL_URL in .env file to the external url of your newly created database or you can setup a new psql database in ubuntu using this [guide](./setupdatabase.md)

After setting up a database 
```
python manage.py makemigrations accounts chat memo
```

```
python manage.py migrate
```

## Setup Static files
This project is currently configured to use amazon s3 bucket, create an amazon s3 bucket and set the value of the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY in your .env file 

After configuring an s3 bucket run 
```
python manage.py collectstatic
```

This will transfer all static files to your amazon bucket 
It may take a few minutes to complete 

## Test gunicorns ability to serve the project 
```
cd /home/<username>/NNRA-circle/nnra_circle
```

```
gunicorn --bind 0.0.0.0:8000 nnra_circle.asgi -w 4 -k uvicorn.workers.UvicornWorker
```

If the step above runs without errors then we are good to go 
`CTRL + c` to exit the running process

#  Creating systemd Socket and Service Files for Gunicorn
Doing this will enable gunicorn to serve our file in the background and also restart itself whenever the system is killed and restarted


Creating the socket file

```
sudo nano /etc/systemd/system/gunicorn.socket
```

Paste in the following 

```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

Save and exit

Next, create and open a systemd service file for Gunicorn 
```
sudo nano /etc/systemd/system/gunicorn.service
```

Paste in the following 

```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=<username>
Group=www-data
WorkingDirectory=/home/<username>/NNRA-circle/nnra_circle
ExecStart=/home/<username>/venv/bin/gunicorn \
          --access-logfile - \
          -k uvicorn.workers.UvicornWorker \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          nnra_circle.asgi:application

[Install]
WantedBy=multi-user.target
```

Replace all `<username>` above with he the username of your server 


## Start and enable the gunicorn socket
```
sudo systemctl start gunicorn.socket
```

```
sudo systemctl enable gunicorn.socket
```

### Confirm the status of the gunicorn socket 

Check for the existence of the gunicorn.sock file within the /run directory:
```
file /run/gunicorn.sock
```

check the status of the socket
```
sudo systemctl status gunicorn.socket
```
=================================
## Incase the socket failed to start 
If the systemctl status command indicated an error occurred or if you do not find the gunicorn.sock file in the directory, this indicates that the Gunicorn socket was not created correctly. Check the Gunicorn socket’s logs by typing:

```
sudo journalctl -u gunicorn.socket
```
Take another look at your /etc/systemd/system/gunicorn.socket and /etc/systemd/system/gunicorn.service files to fix any problems before continuing.

after making any changes to these files retart gunicorn 

```
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```

==============================

Currently, if we've only started the gunicorn.socket unit, the gunicorn.service will not be active yet since the socket has not yet received any connections. we can send a connection to activate it using curl 

```
curl --unix-socket /run/gunicorn.sock localhost
```
This should return a html response 

### Confirm that gunicorn is now active and running 

```
sudo systemctl status gunicorn
```


# Configure nginx to proxy pass go gunicorn 

```
sudo nano /etc/nginx/sites-available/nnra_circle
```

paste the code below 

```
server {
    listen 80;
    server_name 0.0.0.0;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        proxy_pass https://circle-bkt-96.s3.amazonaws.com/;
    }

    location /media/ {
        proxy_pass https://circle-bkt-96.s3.amazonaws.com/media/;
    }

    location /chat/room/ws/ {
        proxy_pass http://unix:/run/gunicorn.sock;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }


    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```


In the above code, server name is the domain we want to serve or project with it should be a value such as `http://example.com` or the ip address of your amazon ec2 instance if you want to connect using ip address if you are using https, then add another listen block under `listen 80;`  and set it to `listen 443`

`https://circle-bkt-96.s3.amazonaws.com/` should be changed to the path to the amazon s3 bucket you created earlier


### Enable the nginx file
```
sudo ln -s /etc/nginx/sites-available/nnra_circle /etc/nginx/sites-enabled
```

You can test your Nginx configuration for syntax errors by typing:
```
sudo nginx -t
```

Update firewall rules 
```
sudo ufw allow 'Nginx Full'
```

Anytime you make changes to the nginx configuration file, you can reload it using 

```
sudo service nginx reload
```

 _Now the site should be available at the domain or ip you specifified as a server_name for nginx_

 ## Seed departments and office tables
 To populate your new database department and office tables with data, make reqeusts to the following endpoints

 ```
http://yourserver/accounts/seed/department/
 ```

```
http://yourserver/accounts/seed/offices/
```

## !!!!🚨🚨🚨 It is imperative that `http://yourserver/accounts/seed/department/` is run before `http://yourserver/accounts/seed/offices/`
