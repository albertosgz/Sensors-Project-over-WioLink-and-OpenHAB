# Sensors-Project-over-WioLink-and-OpenHAB
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
The guide to follow to deploy the server is [here](https://github.com/Seeed-Studio/Wio_Link/wiki/Server%20Deployment%20Guide). But is oriented to linux systems, and we have to install it in a server, with the ports 8080, and so, that the guide uses, already in use. Follow the guide, keeping in mind that:
- The SO is Windows 10 Pro
- The ports should be different
The principal steps, according the guide are set up an Apache instance, with SSL support, and configure it as a reverse proxy. Then install the docker image of the firmware server. And finally connect the app.
We assume next data:
Certificates will name luminosamaintenance.*
Apache plain port: 10080
Apache SSL port: 10443
Firmware plain port: from 19000
Server ip: 192.168.20.37
Install Apache with SSL support
To install the Apache in windows, we decided to install Xampp, because it cames with a console to manage the Apache service, it has a very good integration in Windows and cames to SSL support by default.
Take this site as a reference, from point 2, about how to get running an Apache with SSL support on Windows.
Below the steps to create the certificates.
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

Enable the SSL module.
Open your httpd.conf file (which for me is in C:\Apache\conf\) and uncomment (remove the # sign) the following lines:
#LoadModule ssl_module modules/mod_ssl.so
#Include conf/extra/httpd-ssl.conf

Below the different configuration files needed. In bold the lines modified.
httpd.conf
