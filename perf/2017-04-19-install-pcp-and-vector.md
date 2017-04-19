# Performance Co-Pilot and Vector for Browser Based Metric Visualizations installation

## PCP
```sh
# the PCP install uses a user and group called pcp and they must exist
sudo addgroup pcp
sudo adduser --system --no-create-home --ingroup pcp --shell /bin/false --disabled-password pcp

# installing packages
sudo apt-get update
sudo apt-get install -y git build-essential autoconf flex bison qt4-default qt4-qmake pkg-config libmicrohttpd10 libmicrohttpd-dev

# installing PCP
cd /tmp
git clone git://git.pcp.io/pcp
cd pcp
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var
make
sudo make install

# enable the services
sudo systemctl start pmcd pmwebd
sudo systemctl enable pmcd pmwebd

# see if well configured
pcp
```

## Vector 
### On the fly
```sh
cd /tmp
git clone https://github.com/Netflix/vector.git
cd vector
bower install
npm install
gulp

#Start the web server
cd dist
python2 -m SimpleHTTPServer 8080
```

### More permanent
```sh
sudo mkdir -p /var/www/html
cd /var/www/html
sudo curl -L https://dl.bintray.com/netflixoss/downloads/1.1.0/vector.tar.gz -o vector.tar.gz
sudo tar xvzf vector.tar.gz
sudo service apache2 restart
```

## Resources
[Vector and PCP as used by Netflix](http://techblog.netflix.com/2015/04/introducing-vector-netflixs-on-host.html)
[Redhat (and other)](http://rhelblog.redhat.com/2015/12/18/getting-started-using-performance-co-pilot-and-vector-for-browser-based-metric-visualizations/)
[Vector](http://vectoross.io/)
[PCP](http://pcp.io/documentation.html)
[Linux Perf by Brendan Gregg](http://www.brendangregg.com/linuxperf.html)
