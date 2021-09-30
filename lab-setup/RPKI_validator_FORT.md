# RPKI validator: FORT

------

***Created and tested by Nicolas Antoniello*** (*@nantoniello*)

> (2021-08-11)

*I'd like to thank very much **Santiago Aggio** for his help and documentation about how to set up and configure FORT validator*

------



## Installing and running our own FORT validator

> (this tutorial will get FORT validator up and running in an **Ubuntu Server** fresh install)



### Get all needed packages

```
# apt update
# apt -y --no-install-recommends install \
       autoconf automake build-essential git libjansson-dev \
       libssl-dev pkg-config rsync ca-certificates curl \
       libcurl4-openssl-dev libxml2-dev \
       less vim
```



### Get FORT and compile it

```
# cd
# git clone https://github.com/NICMx/FORT-validator.git
# cd FORT-validator
# ./autogen.sh
# ./configure
# make
# make install
# make clean
```



### Create directories to store FORT config (and other) files

```
# mkdir -p /etc/fort
# mkdir -p /var/fort/tal
# mkdir -p /var/fort/repository
```



### Just in case, get the provided copy of all 5 RIR TALs

```
# cp -rp /root/FORT-validator/examples/tal /var/fort
```



### Now we will update TALs from all 5 RIRs to the latest available

Get TALs and store them in */var/fort/tal* directory

```
# fort --init-tals --tal /var/fort/tal
```



Check /var/fort/tals directory and look for all five RIR TALs files:

```
# ls -larth /var/fort/tal
```

```
-rw-r--r-- 1 root root 496 Aug 11 22:25 afrinic.tal
-rw-r--r-- 1 root root 466 Aug 11 22:25 apnic.tal
-rw-r--r-- 1 root root 487 Aug 11 22:25 arin.tal
-rw-r--r-- 1 root root 502 Aug 11 22:25 lacnic.tal
-rw-r--r-- 1 root root 482 Aug 11 22:25 ripe-ncc.tal
```



If ARIN's TAL is not present, download it (you'll have to accept ARIN's policy in order to be able to download):

```
# curl -k -o /var/fort/tal/arin.tal https://www.arin.net/resources/manage/rpki/arin-rfc7730.tal
```



### Configure FORT (/etc/fort/config.json)

Configuration ***server*** parameters are:

- **mode**: must have the value `server` in order to run FORT validator as RTR server.
- **address**: network address where the server will listen for routers.
- **port**: port or service (see `‘$ man services’`) where the server will listen for routers.
- **backlog**: max number of outstanding connections in the server listen queue.



Configuration ***server.interval*** parameters are:

- **validation**: Time (in seconds) the validator should wait after updating and validating the ROA cache before updating again from the global repository. It also affects on how often the server could send update notifications to the routers (if there are updates as result of the last validation cycle).
- **refresh**: Time (in seconds) the RTR client (router) has to wait before trying to poll the validator cache. The [RFC8210](https://tools.ietf.org/html/rfc8210) default is 3600 seconds.
- **retry**: Time (in seconds) the RTR client should wait before retrying after a failed refresh of the cache. The RFC8210 default is 600 seconds.
- **expire**: Time (in seconds) the RTR client can use its validated ROA cache for if cannot refresh the data, after which it should discard. The RFC8210 default is 7200 seconds.
- **output**: Print validated ROAs to a CSV file.



Configuration file (/etc/fort/config.json) should be something like this:

```
{
	"tal": "/var/fort/tal",
	"local-repository": "/var/fort/repository",
	"rsync-strategy": "root",
	"shuffle-uris": true,
	"mode": "server",

	"server": {
		"port": "323",
		"backlog": 16,
		"interval": {
	            "validation": 900,
	            "refresh": 900,
	            "retry": 600,
	            "expire": 7200
	        }
	},

	"log": {
		"color-output": true,
		"file-name-format": "file-name"
	},

	"rsync": {
		"program": "rsync",
		"arguments-recursive": [
			"--recursive",
			"--times",
			"$REMOTE",
			"$LOCAL"
		],
		"arguments-flat": [
			"--times",
			"--dirs",
			"$REMOTE",
			"$LOCAL"
		]
	},

	"incidences": [
		{
			"name": "incid-hashalg-has-params",
			"action": "ignore"
		}
	],

	"output": {
		"roa": "/var/fort/fort_roas.csv"
	}
}
```



### Make FORT run as a daemon

Let's create our */lib/systemd/system/fortd.service* file:

```
# nano /lib/systemd/system/fortd.service
```

```
[Unit]
Description=FORT_RPKI_validator
After=network.target

[Service]
Type=simple
WorkingDirectory=/usr/local/bin
ExecStart=/usr/local/bin/fort --configuration-file=/etc/fort/config.json
StandardOutput=append:/var/log/fortd.log
StandardError=append:/var/log/fortd.log
SyslogIdentifier=FORT_RPKI_validator
TimeoutStartSec=10
TimeoutStopSec=10
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
Alias=fortd.service
```



And now install and enable our ***fortd*** service:

```
# systemctl daemon-reload
# systemctl enable fortd
# systemctl start fortd
# systemctl status fortd
```

```
● fortd.service - FORT service (fort) - FORT RPKI validator
     Loaded: loaded (/lib/systemd/system/fortd.service; enabled; vendor preset: enabled)
    Drop-In: /run/systemd/system/service.d
             └─zzz-lxc-service.conf
     Active: active (running) since Wed 2021-08-11 23:23:41 UTC; 33min ago
   Main PID: 6807 (fort)
      Tasks: 37 (limit: 19204)
     Memory: 73.4M
     CGroup: /system.slice/fortd.service
             └─6807 /usr/local/bin/fort --configuration-file=/etc/fort/config.json

Aug 11 23:23:41 RPKIfortX systemd[1]: Started FORT service (fort) - FORT RPKI validator.
Aug 11 23:23:41 RPKIfortX fort[6807]: INF: Disabling validation logging on syslog.
Aug 11 23:23:41 RPKIfortX fort[6807]: INF: Disabling validation logging on standard streams.
Aug 11 23:23:41 RPKIfortX fort[6807]: INF: Console log output configured; disabling operation logging on syslog.
Aug 11 23:23:41 RPKIfortX fort[6807]: INF: (Operation Logs will be sent to the standard streams only.)
```

Note that all log from the service will be placed in ***/var/log/fortd.log*** file.

