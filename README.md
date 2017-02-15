# Sensors Project over WioLink and OpenHAB2
Documentation and configuration files to get ready a Monitorization tool using WioLink boards and OpenHAB software

## Overview
Many buildings are not look after during winter due to no availability of workers to take care on them. 
Them, the Maintenance Team needs sensors which can help, through alarms, to take care in case of freeze tubes or water leaks.

## Goals
- Monitor the temperature: Install sensors in inhabitated buildings.
- Check water leaks: Install sensors in places tending to leak.
- Send alerts: Send mail alerts if a threshols is over.
- Control panel: To have a panel to check values and historics.

## Specifications

### Sensors
The system selected to monitor the different situatios is the [wio link board with grove sensors](https://github.com/Seeed-Studio/Wio_Link). We choose them because:
- are open source and open hardware, 
- which allow us to have control and customize them according our needs,
- allow OTA confituration
- have a wide range of sensors and actuators available

## Control Panel
The software who store the values, the historic, and send alerts in case, is [OpenHAB](http://www.openhab.org/).
Is open source and allow many different customization. It has also an [iOS](https://itunes.apple.com/us/app/wio-link/id1054893491?mt=8) and [Android](https://play.google.com/store/apps/details?id=cc.seeed.iot.ap) app.

## Design
```
                   reads               +----------+
     +---------+   periodically        | Sensors  |
     | OpenHAB |---------------------->|  Server  |
     +---------+                       +----------+
         ^ |                                |
read     | | send                           |
historic | | alerts                         |
         | \/                               |
   +------------+                      +----------+
   |    user    |                      |  Sensor  |
   | (web, app) |                      +----------+
   +------------+
```

# Instructions

## Deploy Sensors Firmware server

** Install the Wio Link server in Windows is experimental **

The guide to follow to deploy the server is [here](https://github.com/Seeed-Studio/Wio_Link/wiki/Server%20Deployment%20Guide). But is oriented to linux systems, and we have to install it in a server, with the ports 8080, and so, that the guide uses, already in use. Follow the guide, keeping in mind that:
- The SO is Windows 10 Pro
- The ports should be different
The principal steps, according the guide are set up an Apache instance, with SSL support, and configure it as a reverse proxy. Then install the docker image of the firmware server. And finally connect the app.

We assume next data:
* Certificates will name `luminosamaintenance.*`
* Apache plain port: 10080
* Apache SSL port: 10443
* Firmware plain port: from 19000
* Server ip: 192.168.20.37

### Install Apache with SSL support
To install the Apache in windows, we decided to install Xampp, because it cames with a console to manage the Apache service, it has a very good integration in Windows and cames to SSL support by default.

Take this site as a reference, from point 2, about how to get running an Apache with SSL support on Windows.

Below the steps to create the certificates.

```
Windows PowerShell
Copyright (C) 2016 Microsoft Corporation. All rights reserved.

PS C:\Users\Luminosa Maintenance> cd ..
PS C:\Users> cd ..
PS C:\> cd .\xampp\
PS C:\xampp> cd .\apache\
PS C:\xampp\apache> cd .\bin\


PS C:\xampp\apache\bin> .\openssl req -config openssl.cnf -new -out luminosamaintenance.csr -keyout luminosamaintenance.pem
WARNING: can't open config file: c:/openssl-1.0.2j-win32/ssl/openssl.cnf
Generating a 1024 bit RSA private key
.......................++++++
...................++++++
writing new private key to 'luminosamaintenance.pem'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:US
State or Province Name (full name) []:NY
Locality Name (eg, city) []:Hyde Park
Organization Name (eg, company) []:Mariapolis Luminosa
Organizational Unit Name (eg, section) []:Maintenance Team
Common Name (eg, your websites domain name) []:192.168.20.37
Email Address []:luminosa1.info@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:

PS C:\xampp\apache\bin> .\openssl rsa -in luminosamaintenance.pem -out luminosamaintenance.key
WARNING: can't open config file: c:/openssl-1.0.2j-win32/ssl/openssl.cnf
Enter pass phrase for luminosamaintenance.pem:
writing RSA key

PS C:\xampp\apache\bin> .\openssl x509 -in luminosamaintenance.csr -out luminosamaintenance.cert -req -signkey luminosam
aintenance.key -days 365
WARNING: can't open config file: c:/openssl-1.0.2j-win32/ssl/openssl.cnf
Signature ok
subject=/C=US/ST=NY/L=Hyde Park/O=Mariapolis Luminosa/OU=Maintenance Team/CN=192.168.20.37/emailAddress=luminosa1.info@gmail.com
Getting Private key
PS C:\xampp\apache\bin>
```

Enable the SSL module.
Open your httpd.conf file (which for me is in C:\Apache\conf\) and uncomment (remove the # sign) the following lines:
```
#LoadModule ssl_module modules/mod_ssl.so
#Include conf/extra/httpd-ssl.conf
```
Below the different configuration files needed

#### httpd.conf
In bold the lines modified.
```
Listen 192.168.20.37:10080
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
LoadModule ssl_module modules/mod_ssl.so
ServerAdmin foobar@gmail.com
ServerName 192.168.20.37
```
Here the [link](sample_files/httpd.conf) to the file

#### httpd-ssl.conf
In bold the lines modified.
```
Listen 10443
<VirtualHost _default_:10443>
  ServerName 192.168.20.37:10443
  ServerAdmin foobar@gmail.com
  SSLCertificateFile "conf/ssl.crt/foobar.cert"
  SSLCertificateKeyFile "conf/ssl.key/foobar.key"  

  # WIO LINK SETTINGS
  BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown

  ProxyRequests Off
  ProxyPreserveHost On
  <Proxy *>
    Order deny,allow
    Allow from all
  </Proxy>

  ProxyPass /v1/node/event ws://127.0.0.1:10090/v1/node/event
  ProxyPassReverse /v1/node/event ws://127.0.0.1:10090/v1/node/event

  ProxyPass /v1 http://127.0.0.1:10090/v1
  ProxyPassReverse /v1 http://127.0.0.1:10090/v1
</VirtualHost>                                  

```
Here the [link](sample_files/httpd-ssl.conf) to the file

#### httpd-vhosts.conf
In bold the lines modified.
```
NameVirtualHost 192.168.20.37:10080
<VirtualHost _default_:10080>
    ServerAdmin foobar@gmail.com
    DocumentRoot "C:/xampp/htdocs"
    ServerName 192.168.20.37:10080
    ErrorLog "C:/xampp/apache/logs/error.log"
    CustomLog "C:/xampp/apache/logs/access.log" common
	
	  # WIO LINK SETTINGS
	  ProxyRequests Off
    ProxyPreserveHost On

    ProxyPass /v1/node/resources http://127.0.0.1:10090/v1/node/resources
    ProxyPassReverse /v1/node/resources http://127.0.0.1:10090/v1/node/resources
</VirtualHost>
```

### Unresolved issue connecting to server by http
I can't connect to 10080 port from the app. I have to connect to 10443 instead. I don't know why. But I don't have time to dig into more.

### Deploy firmware server
Install Docker in windows. Follow the [official documentation](https://docs.docker.com/docker-for-windows/). It should be enough with points 1 and 2.
Then come back to the Firmware server deployment guide, and [continue from point 1.2](https://github.com/Seeed-Studio/Wio_Link/wiki/Server%20Deployment%20Guide#12-pull-down-the-image), to pull the image, run it and set it up.
Below the steps run in the shell (in bold the commands run):
```
Windows PowerShell
Copyright (C) 2016 Microsoft Corporation. All rights reserved.

PS C:\Users\Luminosa Maintenance> docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS
NAMES

PS C:\Users\Luminosa Maintenance> docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
78445dd45222: Pull complete
Digest: sha256:c5515758d4c5e1e838e9cd307f6c6a0d620b5e07e6f927b07d05f6d12a1ac8d7
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/

PS C:\Users\Luminosa Maintenance> docker pull killingjacky/wio_link
Using default tag: latest
latest: Pulling from killingjacky/wio_link
75a822cd7888: Pull complete
57de64c72267: Pull complete
4306be1e8943: Pull complete
871436ab7225: Pull complete
37c937b0ca47: Pull complete
608a51124afe: Pull complete
086c59e7b25f: Pull complete
21705250c086: Pull complete
455736e9593d: Pull complete
cf35bb1277e6: Pull complete
115f73e4137a: Pull complete
9554fad1a099: Pull complete
bcd6b85e6d87: Pull complete
cd8528c8f179: Pull complete
d80d6331308a: Pull complete
dc3eeb7c2d04: Pull complete
9e1b3a9ffdf9: Pull complete
58afcb3af68a: Pull complete
a4bd0d1a70fb: Pull complete
d961449e88e2: Pull complete
c397cdbe7906: Pull complete
f564dcfa3931: Pull complete
Digest: sha256:ee267a9c76873d9f427f0c8373b4eafbbff96e702da1e2472e0e1044cd5b7f03
Status: Downloaded newer image for killingjacky/wio_link:latest

PS C:\Users\Luminosa Maintenance> docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
killingjacky/wio_link   latest              be9a21164087        3 weeks ago         886 MB
hello-world             latest              48b5124b2768        3 weeks ago         1.84 kB

PS C:\Users\Luminosa Maintenance> docker run -itd  --name p3 --restart=always -p 127.0.0.1:10090:8080 -p 10091:8081 -p 19000:8000 -p 19001:8001 -v /root/esp8266_iot_node killingjacky/wio_link
a3d0b06d9a325d2c14fad66c20938f182622512a5588d4715722461f26397725

PS C:\Users\Luminosa Maintenance> docker attach p3
root@a3d0b06d9a32:~/esp8266_iot_node#
root@a3d0b06d9a32:~/esp8266_iot_node#
root@a3d0b06d9a32:~/esp8266_iot_node# vim /root/esp8266_iot_node/config.py
root@a3d0b06d9a32:~/esp8266_iot_node# supervisorctl stop esp8266
esp8266: stopped
root@a3d0b06d9a32:~/esp8266_iot_node# supervisorctl start esp8266
esp8266: started
root@a3d0b06d9a32:~/esp8266_iot_node#
PS C:\Users\Luminosa Maintenance> docker ps
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS
                 NAMES
a3d0b06d9a32        killingjacky/wio_link   "/bin/sh -c '/etc/..."   4 minutes ago       Up 4 minutes        0.0.0.0:19000->8000/tcp, 0.0.0.0:19001->8001/tcp, 127.0.0.1:10090->8080/tcp, 0.0.0.0:1
0091->8081/tcp   p3
```

### Access to server from phone app
At this, you should be able to connect from the app, android or iOS. Here there is a little guide.
Just sign up with your mail, a new password, and in the server location, choose Customize Server https://192.168.20.37:10443

E voila'!

## Install OpenHAB
Dowload OpenHAB zip package
http://www.openhab.org/downloads.html

## Configure OpenHAB for windows
http://docs.openhab.org/installation/windows.html

## Install Addons
### Actions
* Mail actions
* Transformations
* JSONPath transformation

### MISC
* OpenHAB Cloud

## Set up the items and rules
TODO

## Change Port of OH2 instance
TODO

## Set up myopenhab.org connection
TODO







