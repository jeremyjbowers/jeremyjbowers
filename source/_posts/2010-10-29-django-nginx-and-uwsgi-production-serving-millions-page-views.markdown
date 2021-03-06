---
layout: post
title: "Django, Nginx and uWSGI in production"
date: 2010-10-29 20:27
comments: true
categories: [Lessons, Varnish, Infrastructure]
---

Upgrading your server stack to Nginx and uWSGI is easier than you'd think. Here's the process I've worked through at the St. Petersburg Times and that we're using in production on several sites like PolitiFact, Mug Shots and HomeTeam. We're now serving millions of page views per year on Nginx and uWSGI, and I'll show you how we're doing it.
<!-- more -->

First, some light reading. Check out Simon Westphal's excellent post "Running Django with Nginx and uWSGI". This post was the framework for our approach, though we've updated some pieces.

Second, the environment. We use (and highly recommend) Rackspace Cloud Servers and Ubuntu 10.04 LTS "Lucid". These instructions presume you have a working installation of Ubuntu 10.04, a working Django project that you're currently serving up some other way, and a reasonable knowledge of getting around in a Linux shell.

Install some software
----------------------

```
sudo apt-get install binutils python-psycopg2 python-setuptools libgeos-3.1.0 libgeos-c1 libgdal1-1.6.0 python2.6 python2.6-dev psmisc subversion git-core mercurial python-imaging locate ntp python-dateutil libxml2-dev libpcre3 libpcre3-dev nginx
```

This is a slew of software to install, but we use this stuff all the time. You've probably got much of this installed on your system by default if you're running Django projects on it now, but it never hurts to be sure. Also, notice that we install Nginx (for the conf files and stuff) even though we're going to replace the binary later.

Also, let's create some directories. At the Times, we use /opt/ for most of our stuff, so I'll demonstrate what we do.

```
mkdir /opt/django-projects/
mkdir /opt/run/
mkdir /opt/lock/
mkdir /opt/log/
sudo chmod -R 777 /opt/run/ /opt/log/ /opt/lock/ /opt/django-projects/
```
Why create shadow versions of /var/log/, /var/lock/ and /var/run/? Simple: Rackspace blitzes user-created files in those directories if you're unfortunate enough to encounter a reboot. Holding those pid, log and lock files in /opt/ is a bit non-standard, but it means you'll never be troubleshooting why Nginx isn't restarting after a reboot.

Get and install Nginx and uWSGI
--------------------------------

```
cd /opt/
wget http://projects.unbit.it/downloads/uwsgi-0.9.6.4.tar.gz
wget http://nginx.org/download/nginx-0.8.51.tar.gz
tar -xzf nginx-0.8.51.tar.gz
tar -xzf uwsgi-0.9.6.4.tar.gz
```

The newer versions of Nginx don't even require compiling a special module for uWSGI -- the support is built-in. This is very nice.
Let's compile and install uWSGI. Wait, you have compiled something before, right? It's really not that bad, I swear. Check this out:

```
cd uwsgi-0.9.6.4
make
cp uwsgi /usr/sbin/
```

Was that really so bad? I didn't think so. The Nginx part of this is a little bit more difficult, but it's not so bad either.

```
cd ../nginx-0.8.51/
touch /opt/run/nginx.pid
touch /opt/lock/nginx.lock
touch /opt/log/nginx/error.log
./configure --conf-path=/etc/nginx/nginx.conf\
--error-log-path=/opt/log/nginx/error.log\
--pid-path=/opt/run/nginx.pid\
--lock-path=/opt/lock/nginx.lock\
--sbin-path=/usr/sbin\
--without-http-cache
make
make install 
```

YMMV: I've compiled this version of Nginx without things like SSL support and other goodies. Check about on the internet if you're interested in a more fully-featured version.

Prep the WSGI interface for your Django project
------------------------------------------------

Presuming you've installed your Django project to /opt/django-projects/foo/, here's the kinds of things you'd need to do to make your Django app WSGI-friendly.

```
cd /opt/django-projects/foo/
mkdir uwsgi
touch uwsgi/__init__.py
nano -w uwsgi/wsgi_app.py
```

Crap. Now everyone knows I'm not a vi user. And that I'm not appropriately 1337 enough to use a nuclear-powered text editor. Anyway, here's the bits you need to include in that wsgi_app.py.

```
#! /usr/bin/env python
import sys
import os
import django.core.handlers.wsgi
sys.path.append('/opt/django-projects/')
os.environ['DJANGO_SETTINGS_MODULE'] = 'foo.settings'
application = django.core.handlers.wsgi.WSGIHandler()
```

Introduce uWSGI to Upstart
---------------------------

Upstart is installed as an init daemon on Ubuntu 10.04, and I much prefer writing Upstart conf files to writing Init scripts. Here's what you'll need to do to make sure that Upstart is familiar with uWSGI.

```
cd /etc/init/
nano -w foo.conf
Creating this foo.conf file in /etc/init/ gives Upstart the information about the uWSGI instances for your project. That's how we'll control uWSGI and your app — through Upstart commands like "sudo service foo restart". Here's what you'll need to put in that foo.conf file.
description "uWSGI server for Project Foo"
start on runlevel [2345]
stop on runlevel [!2345]
respawn
exec /usr/sbin/uwsgi --socket /opt/run/foo.sock\
--chmod-socket --module wsgi_app\
--pythonpath /opt/django-projects/foo/uwsgi\
-p 1
```

The line with exec /usr/sbin/uwsgi demonstrates some options you might be interested in.

*	-p 1 means to run a single uWSGI process. We run ours in production with -p 12 on a 1024 MB app server.
*	-t 15 means to kill any process which takes longer than 15 seconds to execute. Mike Tigas uses --harakiri (same as -t) 30 on his personal site.
*	--logto /opt/foo_uwsgi.log would write an access log to the file foo_uwsgi.log.

There are many more options available at the uWSGI documentation site. We're currently playing around with things like the process reaper and some of the more esoteric buffer/socket options.

You'll need to let Upstart know you've added a new daemon for it with this command.

```
initctl reload-configuration
```

Then, you can do something like this to start your uWSGI instance.
```
sudo service foo start
```

When you make a change that needs your project to reinitialize (like a change to a View or to the settings.py file), you can just do something like this.

```
sudo service foo restart
```

Configuring Nginx
------------------

The last piece of the puzzle is configuring Nginx to talk to uWSGI. Let's talk for a moment about sockets; in the uWSGI configuration above, there's a command-line flag like --socket /opt/run/foo.sock. A socket is a filesystem-level method for two processes to communicate. Most WSGI or FCGI methods use TCP sockets for communication, which introduces the additional overhead of encoding requests and responses into TCP and then decoding them on the other end. Because uWSGI uses UNIX sockets for communication with the Web server (in this case, Nginx), it avoids that overhead and gains a bit of speed.

Newer versions of Nginx have a uWSGI socket interface already built-in — we just need to signal to Nginx that we're using uWSGI and which socket we'd like to pass the request along to. Here's a sample version for you to use in an Nginx configuration file.

```
user www-data;
worker_processes 1;
error_log /opt/log/nginx.log;
pid /opt/run/nginx.pid;
events {
	worker_connections 1024;
	use epoll;
}
http {
	include /etc/nginx/mime.types;
	default_type application/octet-stream;
	access_log /var/log/nginx/access.log;
	keepalive_timeout 65;
	proxy_read_timeout 200;
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	gzip on;
	gzip_min_length 1000;
	gzip_proxied any;
	gzip_types text/plain text/css text/xml
	application/x-javascript application/xml
	application/atom+xml text/javascript;
	proxy_next_upstream error;
	server {
	listen 80;
	server_name www.tampabay.com tampabay.com;
	client_max_body_size 50M;
	root /var/www/ht2;
			location / {
				uwsgi_pass unix://opt/run/preps.sock;
				include uwsgi_params;
			}
	}
}
```

Okay, so let's talk about what's happening here. Feel free to skip this if you're already a master Nginx hacker.

```user www-data;```

Run as the user ww-data.

```worker_processes: 1;```

Use a single worker process. We use up to 3 on some boxes which take very heavy load, but only one on our application servers.

```error_log: /opt/log/nginx.log;```

Log errors to a file in /opt/log/. Touch this file before starting Nginx.

```pid /opt/run/nginx.pid;```

Use a pid file in /opt/run/. Touch this file before starting Nginx.

```
events {
	worker_connections 1024;
	use epoll;
}
```

Use 1024 connections per worker and use epoll as the quarterback for distribution. On some of our boxes, we use up to 10000 connections per worker.

```
include /etc/nginx/mime.types;
default_type application/octet-stream;
access_log /var/log/nginx/access.log;
keepalive_timeout 65;
proxy_read_timeout 200;
sendfile on;
tcp_nopush on;
tcp_nodelay on;
gzip on;
gzip_min_length 1000;
gzip_proxied any;
gzip_types text/plain text/css text/xml
application/x-javascript application/xml
application/atom+xml text/javascript;
proxy_next_upstream error;
```

Much basic configuration baloney that we've gleaned from a million places on the Web. These are basically just sensible defaults. Change as you require.

```
server {
listen 80;
server_name www.tampabay.com tampabay.com;
client_max_body_size 50M;
root /var/www/ht2;
	location / {
		uwsgi_pass unix://opt/run/preps.sock;
		include uwsgi_params;
	}
}
```

Initiate another server and have it listen on port 80. Pass any request along to the socket for our uWSGI project (in this case, preps).
You can start Nginx for the first time by doing something like this.

```
sudo service nginx start
```

You won't need to restart Nginx if you're making a code change, just uWSGI.