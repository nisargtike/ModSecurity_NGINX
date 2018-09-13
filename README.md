# ModSecurity_NGINX
Repo for the installation of NGINX with ModSecurity 

#### Logging in as root
```sudo su```

#### Install required packages.
```yum localinstall Development\ Tools/*```


```
cd "Tools"
yum localinstall *
```

#### copy ModSecurity to /usr/src
```cp -r ./ModSecurity /usr/src

cd ModSecurity
sed -i '/AC_PROG_CC/a\AM_PROG_CC_C_O' configure.ac
sed -i '1 i\AUTOMAKE_OPTIONS = subdir-objects' Makefile.am
./autogen.sh
./configure --enable-standalone-module --disable-mlogc
make
```
#### copy nginx to /usr/src
```cp -r ./nginx /usr/src

groupadd -r nginx
useradd -r -g nginx -s /sbin/nologin -M nginx
```
#### compile Nginx while enabling ModSecurity and SSL modules:
```cd nginx-1.10.3/
./configure --user=nginx --group=nginx --add-module=/usr/src/ModSecurity/nginx/modsecurity --with-http_ssl_module
make
make install
```
#### Modify the default user of Nginx:
```sed -i "s/#user  nobody;/user nginx nginx;/" /usr/local/nginx/conf/nginx.conf ```




#### Test the installation
```/usr/local/nginx/sbin/nginx -t```

#### Following should be the output of the file
```
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```



#### Setup a systemd unit file
```
cat <<EOF>> /lib/systemd/system/nginx.service
[Service]
Type=forking
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
KillStop=/usr/local/nginx/sbin/nginx -s stop

KillMode=process
Restart=on-failure
RestartSec=42s

PrivateTmp=true
LimitNOFILE=200000

[Install]
WantedBy=multi-user.target
EOF
```


#### start/stop/restart nginx
```
systemctl start nginx.service
systemctl stop nginx.service
systemctl restart nginx.service
```



#### Configure ModSecurity and Nginx
```
vi /usr/local/nginx/conf/nginx.conf
```
#### Find the following segment within the http {} segment:

```
location / {
    root   html;
    index  index.html index.htm;
}
```

#### Insert the below lines into the location / {} segment: 
```
ModSecurityEnabled on;
ModSecurityConfig modsec_includes.conf;
#proxy_pass http://localhost:8011;
#proxy_read_timeout 180s;

# The final result should be:
location / {
    ModSecurityEnabled on;
    ModSecurityConfig modsec_includes.conf;
    #proxy_pass http://localhost:8011;
    #proxy_read_timeout 180s;
    root   html;
    index  index.html index.htm;
}
```

#### Save the file
```
:wq!
```



#### Create a file named /usr/local/nginx/conf/modsec_includes.conf
```
cat <<EOF>> /usr/local/nginx/conf/modsec_includes.conf
include modsecurity.conf
include owasp-modsecurity-crs/crs-setup.conf
include owasp-modsecurity-crs/rules/*.conf
EOF
```


#### Import ModSecurity configuration files:
```
cp /usr/src/ModSecurity/modsecurity.conf-recommended /usr/local/nginx/conf/modsecurity.conf
cp /usr/src/ModSecurity/unicode.mapping /usr/local/nginx/conf/
```

#### Modify the /usr/local/nginx/conf/modsecurity.conf file:
```
sed -i "s/SecRuleEngine DetectionOnly/SecRuleEngine On/" /usr/local/nginx/conf/modsecurity.conf
```



#### Add OWASP ModSecurity CRS (Core Rule Set) files:
#### Copy owasp/owasp-modsecurity-crs to /usr/local/nginx/conf/
```
cp -r owasp/owasp-modsecurity-crs /usr/local/nginx/conf/
cd /usr/local/nginx/conf/owasp-modsecurity-crs/
mv crs-setup.conf.example crs-setup.conf
cd rules
mv REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
mv RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```


#### Test ModSecurity
```
systemctl start nginx.service
```

#### Install firewalld and start it
```
yum localinstall firewalld/*
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld
```

#### open port 80
```
firewall-cmd --zone=public --permanent --add-service=http
firewall-cmd --reload
```
#### Go to the following link
```
http://IP-ADDRESS/?param="><script>alert(1);</script>
```

#### Use grep to fetch error messages as follows:
```
grep error /usr/local/nginx/logs/error.log
```


### IMP LINKS

#### NGINX setup ModSecurity

https://www.vultr.com/docs/how-to-install-modsecurity-for-nginx-on-centos-7-debian-8-and-ubuntu-16-04

#### Firewalld

https://www.tecmint.com/fix-firewall-cmd-command-not-found-error/

