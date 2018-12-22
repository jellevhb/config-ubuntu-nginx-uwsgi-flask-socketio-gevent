:wrench: Config for Ubuntu 18.04 + NGINX+ uWSGI + Flask + SocketIO + Gevent
===

I had a hard time finding out on how to get SocketIO to work through NGINX and uWSGI. That's why I have put all needed config files here for future reference. Hopefully, it might help you too.

## Notes

The server is setup using the following guides. You don't have to follow them, they're just here in case you are starting from scratch.
* [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)
* [How To Serve Flask Applications with uWSGI and Nginx on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04)

Some mistakes I have made in the process:
* It's important that your project files are not owned by the `root` user. Otherwise, NGINX will complain later on because it can't access files that need `root` permissions. So make sure you have another user and let him own the project ([chown](https://linux.die.net/man/1/chown))
* Take care of the python version you use. On Ubuntu 18.04, you have to explicitly use the `python3.6` command if you need python 3.6
* To install your dependencies, only use the `pip` command when you are in your virtual environment. This `pip` will install everything in your local environment

## Python dependencies

* flask
* flask-socketio
* uwsgi
* gevent

Gevent is the easiest way to let SocketIO work with the built-in websocket support of uWSGI. Below are the relevant parts of `pip freeze` in my case.

```
Flask==1.0.2
Flask-SocketIO==3.1.1
gevent==1.3.7
uWSGI==2.0.17.1
(...)
```

## uWSGI config

`wsgi.py`
```
from app import app, socketio

if __name__ == '__main__':
    socketio.run(app)
```

`wsgi.ini`
```
[uwsgi]
module = wsgi:app

master = true
gevent = 500
buffer-size=32768 # optionally
http-websockets = true

socket = myproject.sock
chmod-socket = 660
vacuum = true

die-on-term = true
```

The argument `gevent` is the number of async cores to spawn ([Gevent docs](https://uwsgi-docs.readthedocs.io/en/latest/Gevent.html)). When you raise this number, you can get an error saying something like `invalid request block size: 21327 (max 4096)...skip`. To fix this, add `buffer-size=32768` to the config ([source](https://stackoverflow.com/a/26941287))

## NGINX config

`/etc/nginx/sites-available/myproject`
```
server {
    listen 80;
    server_name example.com;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/path/to/project/myproject.sock;
    }

    location /socket.io/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        include uwsgi_params;
        uwsgi_pass unix:/path/to/project/myproject.sock;
    }
}
```

Both `location` sections use `uwsgi_pass` directly to the unix socket. It's important to add `include uwsgi_params` in both sections.

## systemd service (optional)

Making a systemd service is optional. You can start uWSGI the way you like.

`/etc/systemd/system/myproject.service`
```
[Unit]
Description=uWSGI instance to serve myproject
After=network.target

[Service]
User=myuser
Group=www-data
WorkingDirectory=/path/to/project
Environment="PATH=/path/to/project/venv/bin"
ExecStart=/path/to/project/venv/bin/uwsgi --ini wsgi.ini

[Install]
WantedBy=multi-user.target
```
