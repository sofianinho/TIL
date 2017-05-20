# From 0 to https for your Flask app with let's encrypt and apache2 on Ubuntu 16.04

After you finished your awesome flask app, whether it is an API or a dynamic website, you are probably thinking you should activate an HTTPS planefor your users. You could generate your very own self-signed certificate and doing something like [this](http://flask.pocoo.org/snippets/111/): 
```
from OpenSSL import SSL
context = SSL.Context(SSL.SSLv23_METHOD)
context.use_privatekey_file('yourserver.key')
context.use_certificate_file('yourserver.crt')
...

app.run(host='127.0.0.1',port='12344', debug = False/True, ssl_context=context)
```
You should probably also know that openssl and pyopenssl (especially in the form above) is not recommended for production grade deployments. This also means that your future clients will either have to install your self generated certificate (not trivial for the vast majority) or be dommed to have that ugly crossed padlock in the address bar after clicking on the "exception" button of the webbrowser. It is worse for CLI tools like `curl`, where you have to add an `--insecure` switch to your command or a `git config http.sslVerify false` for your development environment.

## Knowing your options
The free ones obviously. 

At this stage, you know that what you need is a certificate signed by a trusted authority. This is what you had with the self-signed certificate (the trusted authority being yourself). Now, you need one that is globally trusted. Some of them are listed in your browser (Firefox: settings -> Advanced -> Certificates -> View Certificates -> Authorities), or in your Linux (Ubuntu: /usr/share/ca-certificates/mozilla/).

- Beame-insta-ssl. The obtained certificate is verified by 'Beame.io Ltd'. You have to install the nodejs tool, and chose between terminating the secure connection at beamio servers (insecure) or terminate it locally in your server (better). The beamio tool works similarly to ngrok, with tunneling/proxying a remote location with your local server port, you obtain a globally reachably URL in the process (nice!). On the down side, I did not like the instability of the proxy application (beame-insta-ssl) that was crashing at every attempt to join my URL. I will definitely give it a try in a few months if it is more stable.

- Let's Encrypt gives you your very own certficate with associated tools (such as `certbot`). The certbot application needs global visibility and reachability of port 443 on your server in order to succeed in the challenge/response operation necessary to validate your certificate. If you are like me, doing your development at home or in a lab, you need to do the right port forwardings in your Internet Box (CPE). If you are in a corporate environment, you should have a chat with your network administrator. If you have a VPS somewhere at AWS, Heroku, CF, or whatever, you should review your security policies for the Igress/Egress connections. If you know your public IP address already, you can use this `A.B.C.D.xip.io` URL as your global URL. If you don't know your global adress, or simply prefer (like me), you can use `duckdns.org` to obtain 5 free domain names with the `.duckdns.org` suffix. The URL is necessary because Let's Encrypt (righfully so) does not accept IP addresses as IDs for certicates (which is the standard).

- I also poked around with sslip.io, only to discover that it was no longer supported because of the not controlled redistribution of their certificate/key to whoever wished on their website. 

## Let's Encrypt
### Get you a working apache2 server with mod_wsgi module
```
sudo apt-get install apache2 ssl-cert libapache2-mod-wsgi ca-certificates
```
### Get you a certbot
I prefer the version that is not tied to any server. 
```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot 
```

### Get your certificate
Assuming you already have a domain at duckdns installed:
```
sudo certbot certonly --standalone -d exampledomain.duckdns.org
```
If the operation finishes successfully, you now have your very own brand new certficate/key under `/etc/letsencrypt/live/exampledomain.duckdns.org/`

### Let's apache2 
- Activate the ssl module
```
sudo a2enmod ssl
```
- (Optional) Add a global ServerName in your server conf (so apache won't complain) in file `/etc/apache2.conf`
```
ServerName	exampledomain.duckdns.org
```
- Write a configuration file for your flask app under `/etc/apache2/sites-available`. You can start by copying the `default-ssl.conf` file into `app.conf`. Here is a sample of app.conf:
```
<IfModule mod_ssl.c>
         <VirtualHost _default_:443>
                 ServerName exampledomain.duckdns.org
                 WSGIDaemonProcess app user=www-data group=www-data threads=5 home=/var/www/app/
                 WSGIScriptAlias / /var/www/app/app.wsgi
                 ErrorLog ${APACHE_LOG_DIR}/error.log
                 CustomLog ${APACHE_LOG_DIR}/access.log combined
                 SSLEngine on
                 SSLCertificateFile      /etc/letsencrypt/live/exampledomain.duckdns.org/fullchain.pem
                 SSLCertificateKeyFile   /etc/letsencrypt/live/exampledomain.duckdns.org/privkey.pem
                 SSLCertificateChainFile /etc/letsencrypt/live/exampledomain.duckdns.org/fullchain.pem
                 <directory /var/www/app>
                         WSGIProcessGroup app
                         WSGIApplicationGroup %{GLOBAL}
                         WSGIScriptReloading On
                         Order deny,allow
                         Allow from all
                 </directory>
                 <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                 SSLOptions +StdEnvVars
                 </FilesMatch>   
                 <Directory /usr/lib/cgi-bin>
                                 SSLOptions +StdEnvVars
                 </Directory>    
         </VirtualHost>
</IfModule>

```
- Put your python flask app in the folder indicatd in the configuration (/var/www/app/). You need to at least have the following content:
```
.
├── app.py
├── app.wsgi
└── __init__.py
```
- Put the following content in your `app.wsgi` file:
```python
import sys
     
sys.path.append('/var/www/app')
     
from app import app as application
```
- Activate your newly edited website/app and reload your server:
```
sudo a2ensite app
sudo service apache2 reload
```
- Check your apache2 server status
```
sudo service apache2 status
```
If any errors arise at this stage, you have to read /var/log/apache2/error.log to have a clue of what it could be. My top guesses if this happens: incorrectly configured python environment (pip dependency or a bad virtualenv), an indorrect certficate path or permissions, a line indicating the domain name resolution under /etc/hosts. These are just blind hypothetical guesses, your personal judgment should prevail here.

- This should work now, and you can normally access your server at your `https://exampledomain.duckdns.org` url with a nice green padlock on your browser. Your CLI tools should work as well. Congrats.

### What is this wsgi that I had to add to my flask app ?
WSGI stands for Web Server Gateway Interface and it is the protocol/language used between web servers and python applications. This [blog entry](http://django-easy-tutorial.blogspot.fr/2017/03/django-request-lifecycle.html?m=1) explains it rather well. The thing is Flask comes with its own development server: Werkzeug. So does Django with its own wsgi server. This abstracts this need when you do your development, but it show up once you deploy and have to use SSL, for instance. 
For further information about Flask applications deployment read further on the [official documentation](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/).  


## Note:
- The above commands and recommendations are also compatible for other frameworks like django.
- The same could be easily applied to nginx.
- I used the same procedure a day before this post (19th of Mai), with no luck. I kept getting a "Timeout" message related to some acme-v1-api registration procedure. Naturally, I thought it was the port forwarding on my Internet Box (CPE) that was not working for some reason. After some googling around, I found out Let's Encrypt were having major problems with their datacenter provider OVH. So think about it if you ever have registration/renewal certifcates operations unexplicably pending until timeout. Here is the link to follow any let's encrypt major announcements related to their API services: http://letsencrypt.status.io/.
