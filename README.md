# time-tracker

Be nilenso's time tracking tool.

## Production Installation
Please use Ubuntu 16.04 LTS.

SSH into your box. Ensure that you are root.

First, apt install all the things.
``` bash
apt install nginx npm node git unzip openjdk-8-jdk postgresql
```

Configure nginx.
``` bash
cd /etc/nginx/sites-available/
mv default default.backup
```

Create a file called `default` in `/etc/nginx/sites-available/` with the following
content:
```
##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# http://wiki.nginx.org/Pitfalls
# http://wiki.nginx.org/QuickStart
# http://wiki.nginx.org/Configuration
#
# Generally, you will want to move this file somewhere, and start with a clean
# file but keep this around for reference. Or just disable in sites-enabled.
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# Default server configuration
#
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	# SSL configuration
	#
	# listen 443 ssl default_server;
	# listen [::]:443 ssl default_server;
	#
	# Note: You should disable gzip for SSL traffic.
	# See: https://bugs.debian.org/773332
	#
	# Read up on ssl_ciphers to ensure a secure configuration.
	# See: https://bugs.debian.org/765782
	#
	# Self signed certs generated by the ssl-cert package
	# Don't use them in a production server!
	#
	# include snippets/snakeoil.conf;

	root /var/www/;

	# Add index.php to the list if you are using PHP
	index index.html;

	server_name _;

	location /api/timers/ws-connect/ {
		# Websockets tunneling
		proxy_pass http://127.0.0.1:8000;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "Upgrade";
	}
	location /api/ {
		# Proxy to backend
		proxy_pass http://127.0.0.1:8000;
	}
	location / {
		# Serve the contents of index.html, no matter what the URL
		try_files $uri /index.html;
	}
}
```

Run
```
nginx -s reload
```

Now, build and deploy the frontend. If you are deploying a branch other than
`master` then be sure to `wget` the correct zip file or `git clone` the correct
branch.

``` bash
wget https://github.com/nilenso/time-tracker-web/archive/master.zip
unzip master.zip
# cd into the source directory
cd time-tracker-web-master/
npm install
# Make sure that REACT_APP_CLIENT_ID is set to your Google client ID
export REACT_APP_CLIENT_ID=your_google_client_id;
nodejs scripts/build.js
cd build
cp -R * /var/www
cd /var/www
rm -rf html/
```

At this point, `nginx` should be serving the frontend. It won't work until
the backend is fully set up.

Create a user to run the backend.
``` bash
adduser timetracker
```

Next, create the postgres database.
``` bash
# This will prompt for a password
sudo -u postgres createuser timetracker -P
sudo -u postgres createdb timetracker -O timetracker
```

Clone/download the backend code. Make sure that you are on the branch that you
want to deploy.
``` bash
su timetracker
cd
git clone https://github.com/nilenso/time-tracker.git
cd time-tracker/
git branch desired-branch
git pull
cd
```

Create a file in `/home/timetracker` called
`env` with the following contents (insert your details as necessary):
``` bash
GOOGLE_TOKENINFO_URL="https://www.googleapis.com/oauth2/v3/tokeninfo"
CP_MAX_IDLE_TIME_EXCESS_CONNECTIONS="1800"
CP_MAX_IDLE_TIME="10800"
DB_CONNECTION_STRING="jdbc:postgresql://localhost/timetracker?user=timetracker&password=your_password_if_any"
GOOGLE_CLIENT_ID="your_google_client_id"
APP_LOG_LEVEL="debug"
PORT="8000"
ALLOWED_HOSTED_DOMAIN="yourgoogleappsforworkhosteddomain.com"
LOG_FILE_PREFIX="/path/to/log/file.log"
```

Install Leiningen to build the project.
``` bash
su timetracker
cd
wget https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein
chmod a+x ./lein
./lein
```

Run the migrations. You should see an INFO log telling you which migrations were
applied if the environment variables are configured correctly.

``` bash
su timetracker
cd ~/time-tracker
../lein deps
../lein migrate
```

Configuring iptables:
``` bash
iptables -A INPUT -i lo -p tcp -m tcp --dport 8000 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 8000 -j DROP
ip6tables -A INPUT -i lo -p tcp -m tcp --dport 8000 -j ACCEPT
ip6tables -A INPUT -p tcp -m tcp --dport 8000 -j DROP
```

`systemd` unit file configuration.
`/etc/systemd/system/timetracker.service`
```
[Unit]
Description=time tracker

[Service]
Type=simple
User=timetracker
ExecStart=/usr/bin/java -jar /home/timetracker/time-tracker.jar
EnvironmentFile=/home/timetracker/env
Restart=always

[Install]
WantedBy=default.target
```

Finally, run `bin/deploy.sh` as root.
``` bash
sudo bin/deploy.sh
```

## License

Copyright © 2016

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
