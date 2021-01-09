# MQTT broker with Mosquitto and Node-Red

## Overview

This is a instuction how to install and configure a Raspberry Pi as an MQTT broker. The free software packages Mosquitto and Node-Red will be used. This project is intended to install a local MQTT broker in a local network which can also be accessed from Internet over a secure TLS connection. The benefit is to have full control over the broker and its availability and reliability instead of using a public (Chinese) broker.

## Prerequisites

* a DynDns-account (e.g. https://freemyip.com/) needs to be created to be able to access your MQTT broker from Internet. Due to that your Internet provider assigns from time to time a different IP-address to your router, your MQTT broker can be accessed allways under the same domain name. If you intent to use your MQTT broker only in the internal network then you can skip this step. As a DDNS-client I could recommend the `ddclient` daemon which also can run on the Raspberry Pi.

* port forwarding needs to be configured in your router for port 8883. This the secured MQTT TLS port.

* a static IP-address needs to be assigned to the Raspberry Pi instead of DHCP. This is required by the router for port forwarding and by the MQTT clients.

## Installation

The descision was made on the broker software Mosquitto.
```
sudo apt update
sudo apt-get install mosquitto mosquitto-clients openssl
```
You’ll have now a very basic and working MQTT broker on port 1883 with no user authentication. 
## Mosquitto daemon

Several commands exist to control the daemon.
```
sudo systemctl stop mosquitto
sudo systemctl start mosquitto
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
```

## Create a user 

I locked down my broker so that only allow those clients who know the password can publish to a topic. You can get super granular here where certain usernames can publish to certain topics only. For my sake I only have 1 user who can publish. Create a password for publishing with:

```
sudo mosquitto_passwd -c /etc/mosquitto/passfile <your username>
```
## The Mosquitto configuration file

The Mosquitto configuration file can be edited with following command. Make sure you have no empty spaces at the end of those lines or Mosquitto may give you an error.
```
sudo nano /etc/mosquitto/mosquitto.conf
```

The Mosquitto configuration file used for this setup you see in the following. Short explanation: anonymous users are not allowed to connect, default internal listener at port 1883 (no TLS), external listener at port 8883 (TLS v1.2 secured) with the required crypto material, no client certificates required.
```
pid_file /var/run/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d

allow_anonymous false
password_file /etc/mosquitto/passfile

listener 1883

listener 8883
certfile /etc/mosquitto/certs/server.crt
cafile /etc/mosquitto/ca_certificates/ca.crt
keyfile /etc/mosquitto/certs/server.key
tls_version tlsv1.2
require_certificate false
```

## Creating TLS crypto material with openssl

1. As we are our own CA (certificate authority) we create first our CA key pair. Make sure this CA key is stored secure that it cannot be stolen.
```
sudo openssl genrsa -out ca.key 2048
```

2. Now create a self-signed certificate for the CA using the CA key. Fill in every fields of the certificate request some information.
```
sudo openssl req -new -x509 -days 15000 -key ca.key -out ca.crt
```

3. Now we create a server key pair that will be used by the broker.
```
sudo openssl genrsa -out server.key 2048
```

4. Now we create a server certificate request. When filling out the form the `Common Name` is important and is usually the domain name of the server. In my case it would be `<user>.freemyip.com`. Please fill in the other fields slightly different values as in step 2.
```
sudo openssl req -new -out server.csr -key server.key
```

5. Now we use the CA key to verify and sign the server certificate. This creates the server.crt file.
```
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 15000
```

Now copy server.crt and server.key to /etc/mosquitto/certs/ and ca.crt to /etc/mosquitto/ca_certificates/. Now changing the owner of these files needs to be done.
```
sudo chown -R mosquitto: /etc/mosquitto/certs/
sudo chown -R mosquitto: /etc/mosquitto/ca_certificates/
```
## Testing the broker


