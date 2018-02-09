# Performance Co-Pilot and Vector for Browser Based Metric Visualizations installation
Install PCP on every server you want to observe. Install Vector only once, where it can reach said servers (by IP or hostname). Also make sure appropriate ports are open/forwarded correctly to pmcd and pmwebd services in the servers (default: pmcd:44321, pmwebd:56676).

For full installation of pcp and utilities (in particular pmwebd), several dependencies need to be satisified. To know them, use this command from the repo bellow
```sh
git clone https://github.com/performancecopilot/pcp
./qa/admin/check-vm
```
A very exhaustive list, not everything is mandatory though. Personnally, I installed on ubuntu 16:04:
```terminal
libpapi-dev libpfm4-dev libboost-dev gdb autoconf libmicrohttpd-dev python-all-dev python3-all-dev libpcp3
```

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
git clone https://github.com/performancecopilot/pcp
cd pcp
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --with-webapi
make
sudo make install

# enable the services
sudo systemctl start pmcd pmwebd
sudo systemctl enable pmcd pmwebd

# see if well configured
pcp
```
### Important
If after all this, you did not find pmwebd (important, it's the API part for vector and other tools), you should (up to you) copy the following script which is a dump from a machine I installed pcp successfully on. The script belongs in /etc/init.d/pmwebd, you must do the initial steps first.

```sh
#! /bin/sh
#
# Copyright (c) 2013-2015 Red Hat.
# Copyright (c) 2005 Silicon Graphics, Inc.  All Rights Reserved.
# 
# This program is free software; you can redistribute it and/or modify it
# under the terms of the GNU General Public License as published by the
# Free Software Foundation; either version 2 of the License, or (at your
# option) any later version.
# 
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY
# or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
# for more details.
# 
# Start or Stop the Performance Co-Pilot (PCP) web daemon for pmcd
#
# The following is for chkconfig on RedHat based systems
# chkconfig: 2345 95 05
# description: pmwebd is the web frontend for the Performance Co-Pilot (PCP)
#
# The following is for insserv(1) based systems,
# e.g. SuSE, where chkconfig is a perl script.
### BEGIN INIT INFO
# Provides:       pmwebd
# Required-Start: $remote_fs
# Should-Start: $local_fs $network $syslog $time $pmcd
# Required-Stop: $remote_fs
# Should-Stop:  $local_fs $network $syslog $pmcd
# Default-Start:  2 3 4 5
# Default-Stop:   0 1 6
# Short-Description: Control pmwebd (the web frontend for PCP)
# Description:       Configure and control pmwebd (the web frontend daemon for the Performance Co-Pilot)
### END INIT INFO

. $PCP_DIR/etc/pcp.env
. $PCP_SHARE_DIR/lib/rc-proc.sh

PMWEBD=$PCP_BINADM_DIR/pmwebd
PMWEBDOPTS=$PCP_PMWEBDOPTIONS_PATH
RUNDIR=$PCP_LOG_DIR/pmwebd
pmprog=$PCP_RC_DIR/pmwebd
prog=$PCP_RC_DIR/`basename $0`

tmp=`mktemp -d /var/tmp/pcp.XXXXXXXXX` || exit 1
status=1
trap "rm -rf $tmp; exit \$status" 0 1 2 3 15

if [ $pmprog = $prog ]
then
    VERBOSE_CTL=on
else
    VERBOSE_CTL=off
fi

case "$PCP_PLATFORM"
in
    mingw)
	# nothing we can usefully do here, skip the test
	#
	;;

    *)
	# standard Unix/Linux style test
	#
	ID=id
	test -f /usr/xpg4/bin/id && ID=/usr/xpg4/bin/id

	IAM=`$ID -u 2>/dev/null`
	if [ -z "$IAM" ]
	then
	    # do it the hardway
	    #
	    IAM=`$ID | sed -e 's/.*uid=//' -e 's/(.*//'`
	fi
	;;
esac

_shutdown()
{
    # Is pmwebd running?
    #
    _get_pids_by_name pmwebd >$tmp/tmp
    if [ ! -s $tmp/tmp ]
    then
	[ "$1" = verbose ] && echo "$pmprog: pmwebd not running"
	return 0
    fi

    # Send pmwebd a SIGTERM, which is noted as a pending shutdown.
    # When finished the currently active request, pmwebd will close any
    # connections and then exit.
    # Wait for pmwebd to terminate.
    #
    pmsignal -p -s TERM pmwebd > /dev/null 2>&1
    $ECHO $PCP_ECHO_N "Waiting for pmwebd to terminate ...""$PCP_ECHO_C"
    gone=0
    for i in 1 2 3 4 5 6
    do
	sleep 3
	_get_pids_by_name pmwebd >$tmp/tmp
	if [ ! -s $tmp/tmp ]
	then
	    gone=1
	    break
	fi

	# If pmwebd doesn't go in 15 seconds, SIGKILL and sleep 1 more time
	# to allow any clients reading from pmwebd sockets to fail so that
	# socket doesn't end up in TIME_WAIT or somesuch.
	#
	if [ $i = 5 ]
	then
	    $ECHO
	    echo "Process ..."
	    $PCP_PS_PROG $PCP_PS_ALL_FLAGS >$tmp/ps
	    sed 1q $tmp/ps
	    for pid in `cat $tmp/tmp`
	    do
		$PCP_AWK_PROG <$tmp/ps "\$2 == $pid { print }"
	    done
	    echo "$prog: Warning: Forcing pmwebd to terminate!"
	    pmsignal -a -s KILL pmwebd > /dev/null 2>&1
	else
	    $ECHO $PCP_ECHO_N ".""$PCP_ECHO_C"
	fi
    done
    if [ $gone != 1 ]	# It just WON'T DIE, give up.
    then
	echo "Process ..."
	cat $tmp/tmp
	echo "$prog: Warning: pmwebd won't die!"
	exit
    fi
    $RC_STATUS -v 
    $PCP_BINADM_DIR/pmpost "stop pmwebd from $pmprog"
}

_usage()
{
    echo "Usage: $pmprog [-v] {start|restart|condrestart|stop|status|reload|force-reload}"
}

while getopts v c
do
    case $c
    in
	v)  # force verbose
	    VERBOSE_CTL=on
	    ;;
	
	*)
	    _usage
	    exit 1
	    ;;
    esac
done
shift `expr $OPTIND - 1`

if [ $VERBOSE_CTL = on ]
then				# For a verbose startup and shutdown
    ECHO=$PCP_ECHO_PROG
else				# For a quiet startup and shutdown
    ECHO=:
fi

if [ "$IAM" != 0 -a "$1" != "status" ]
then
    if [ -n "$PCP_DIR" ]
    then
	: running in a non-default installation, do not need to be root
    else
	echo "$prog:"'
Error: You must be root (uid 0) to start or stop the PCP pmwebd daemon.'
	exit
    fi
fi

# First reset status of this service
$RC_RESET
 
# Return values acc. to LSB for all commands but status:
# 0 - success
# 1 - misc error
# 2 - invalid or excess args
# 3 - unimplemented feature (e.g. reload)
# 4 - insufficient privilege
# 5 - program not installed
# 6 - program not configured
#
# Note that starting an already running service, stopping
# or restarting a not-running service as well as the restart
# with force-reload (in case signalling is not supported) are
# considered a success.
case "$1" in

  'start'|'restart'|'condrestart'|'reload'|'force-reload')
	if [ "$1" = "condrestart" ] && ! is_chkconfig_on pmwebd
	then
	    status=0
	    exit
	fi

	_shutdown quietly

	# pmwebd messages should go to stderr, not the GUI notifiers
	#
	unset PCP_STDERR

	if [ -x $PMWEBD ]
	then
	    if [ ! -f $PCP_PMWEBDCONF_PATH ]
	    then
		echo "$prog:"'
Error: pmwebd control file "$PCP_PMWEBDCONF_PATH" is missing, cannot start pmwebd.'
		exit
	    fi
	    # create directory housing daemon pid file
	    if [ ! -d "$PCP_RUN_DIR" ]
	    then
		mkdir -p -m 775 "$PCP_RUN_DIR"
		chown $PCP_USER:$PCP_GROUP "$PCP_RUN_DIR"
	    fi
	    # create directory which will serve as cwd
	    if [ ! -d "$RUNDIR" ]
	    then
		mkdir -p -m 775 "$RUNDIR"
		chown $PCP_USER:$PCP_GROUP "$RUNDIR"
	    fi
	    cd "$RUNDIR"

	    # salvage the previous versions of any pmwebd
	    #
	    if [ -f pmwebd.log ]
	    then
		rm -f pmwebd.log.prev
		mv pmwebd.log pmwebd.log.prev
	    fi

	    $ECHO $PCP_ECHO_N "Starting pmwebd ..." "$PCP_ECHO_C"

            # Decide whether this is an old- or new-style configuration
            # file, judging by the presence of the OPTIONS= string.
            if grep -q OPTIONS= $PMWEBDOPTS; then
                # Source options files, allow $var expansions etc.
                . $PMWEBDOPTS
            else
                # old options file processing ...
                # only consider lines which start with a hyphen
                # get rid of the -f option
                # ensure multiple lines concat onto 1 line
                OPTIONS=`sed <$PMWEBDOPTS 2>/dev/null \
                                -e '/^[^-]/d' \
                                -e 's/^/ /' \
                                -e 's/$/ /' \
                                -e 's/ -f / /g' \
                                -e 's/^ //' \
                                -e 's/ $//' \
                        | tr '\012' ' ' `

                # formerly implicit
                OPTIONS="$OPTIONS -l pmwebd.log"

                # environment stuff
                #
                eval `sed -e 's/"/\\"/g' $PMWEBDOPTS \
                | awk -F= '
     BEGIN                  { exports="" }
     /^[A-Z]/ && NF == 2    { exports=exports" "$1
                              printf "%s=${%s:-\"%s\"}\n", $1, $1, $2
                            }
     END                    { if (exports != "") print "export", exports }'`
            fi

	    $PMWEBD $OPTIONS &
	    $RC_STATUS -v

	    $PCP_BINADM_DIR/pmpost "start pmwebd from $pmprog"

	    # finally, stop here if running in a container
	    [ -z "$PCP_CONTAINER_IMAGE" ] || exec $PCP_BINADM_DIR/pmpause
	fi
	status=0
        ;;

  'stop')
	_shutdown
	status=0
        ;;

  'status')
	# NOTE: $RC_CHECKPROC returns LSB compliant status values.
	$ECHO $PCP_ECHO_N "Checking for pmwebd:" "$PCP_ECHO_C"
        if [ -r /etc/rc.status ]
        then
            # SuSE
            $RC_CHECKPROC $PMWEBD
            $RC_STATUS -v
            status=$?
        else
            # not SuSE
            $RC_CHECKPROC $PMWEBD
            status=$?
            if [ $status -eq 0 ]
            then
                $ECHO running
            else
                $ECHO stopped
            fi
        fi
	;;

  *)
	[ -z "$PCP_CONTAINER_IMAGE" ] || exec "$@"
	_usage
        ;;
esac
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

### Docker
```sh
docker run   -d   --name vector   -p 8081:80   netflixoss/vector:latest
```

## Resources
[Vector and PCP as used by Netflix](http://techblog.netflix.com/2015/04/introducing-vector-netflixs-on-host.html)

[Redhat (and other)](http://rhelblog.redhat.com/2015/12/18/getting-started-using-performance-co-pilot-and-vector-for-browser-based-metric-visualizations/)

[Vector](http://vectoross.io/)

[PCP](http://pcp.io/documentation.html)

[Linux Perf by Brendan Gregg](http://www.brendangregg.com/linuxperf.html)
