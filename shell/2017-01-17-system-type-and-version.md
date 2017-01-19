# Determine the type and the version of a linux kernel and the distribution 

### Any linux (as far as I tested)
```sh
#The distribution
cat /etc/issue
#The kernel --equivalent to uname -a if you do the 3 commands
cat /proc/sys/kernel/ostype
cat /proc/sys/kernel/osrelease
cat /proc/sys/kernel/version
```
### The debian-based systems
```sh
#Equivalent to lsb_release -a (with a less detail)
cat /etc/debian-version
cat /etc/lsb-release
```

### The redhat-based systems
```sh
# the contents of the files: /etc/{centos-release, redhat-release, os-release}
# equivalent to rpm --query centos-release (for example)
cat /etc/*elease
```
