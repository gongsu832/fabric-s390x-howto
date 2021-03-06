Hyperledger code now builds out of the box on Linux s390x. But you need to have the correct environment, e.g., docker, rocksdb, etc., setup first. Please follow the steps below to setup the environment and build Hyperledger. The instructions assume a Debian host but should also work on Ubuntu. When appropriate, commands for RHEL or SLES are also given.

####1. Install docker
For RHEL and SLES, you can get it at https://www.ibm.com/developerworks/linux/linux390/docker.html. And for Debian, you can get it here at https://github.com/gongsu832/docker-s390x-debian (statically linked binaries should work on all distros). On Debian and Ubuntu, you may need to install aufs-tools if it's not already installed and you want to use aufs (not recommended) as the storage driver..

   ```
   apt-get install aufs-tools
   ```

Start the docker daemon and pull the debian base image.

   ```
   # docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock > /var/log/docker.log 2>&1 &
   # docker pull s390xlinux/debian-base
   ```
   
Note that -H tcp://0.0.0.0:2375 is not really needed for pulling images. It's for doing the Hyperledger "behave" tests later.
   
####2. Install go 1.6
Due to legal reasons, IBM cannot yet distribute prebuilt go 1.6 binary for Linux s390x. There are instructions on how to build one yourself from the publically available source but the process is rather involved. Instead, the easiest way is to get a copy that someone else already built.
   
   ```
   # docker pull brunswickheads/golang-1.6-s390x
   # container_id=$(docker create brunswickheads/golang-1.6-s390x /bin/bash)
   # docker cp $container_id:/usr/local/go - | tar x -C /opt --no-same-owner --no-same-permissions
   # docker rm $container_id
   ```

This will extract go 1.6 from the docker image into /opt/go. To use it, set GOROOT=/opt/go and add /opt/go/bin to your PATH. You can delete brunswickheads/golang-1.6-s390x image afterwards if you no longer need it.
   
####3. Install rocksdb
The example below clones into /opt/rocksdb. You can choose wherever you prefer.

   ```
   # cd /opt
   # git clone https://github.com/facebook/rocksdb.git
   # cd rocksdb
   # make install-shared
   # ldconfig -v
   ```
   
This installs rocksdb library files in /usr/local/lib and header files in /usr/local/include. Inspect the output of "ldconfig -v" and make sure /usr/local/lib is in the searched paths. If not, create a file such as local.conf in /etc/ld.so.conf.d with a line /usr/local/lib in it and run "ldconfig -v" again.
   
####4. Build Hyperledger
Once again, the example below uses GOPATH=/opt/openchain and you are free to use wherever you prefer.

   ```
   # apt-get install libbz2-dev libsnappy-dev zlib1g-dev
   # mkdir -p /opt/openchain/src/github.com/hyperledger
   # cd /opt/openchain/src/github.com/hyperledger
   # git clone https://github.com/s390xlinux/fabric.git
   # cd fabric
   # docker tag $(docker images|grep s390xlinux/debian-base|awk '{print $3}') hyperledger/fabric-baseimage:$(uname -m)-$(cat ./images/base/release)
   # export GOPATH=/opt/openchain
   # make peer
   ```
   
   For RHEL, replace apt-get command with
   ```
   # yum install bzip2-devel snappy-devel zlib-devel
   ```
   
   For SLES, replace apt-get command with
   ```
   # zypper install libbz2-devel snappy-devel zlib-devel
   ```
   
This generates "peer" binary in the build/bin directory.

To build the Hyperledger CA server, which is required when security is enabled,

   ```
   # make membersrvc
   ```

This generates "membersrvc" binary in the build/bin directory.
 
#####To run the unit tests,
   
   ```
   # make unit-test
   ```
   
You should have no FAIL for any test. Note that to rerun the tests, you need to clean up all the leftover containers and images except hyperledger/* images.
   
#####To run the behave tests,

Install docker-compose and behave if you haven't already,

   ```
   # apt-get install python-setuptools python-pip
   # pip install docker-compose behave
   ```

   For RHEL
   ```
   # yum install python-setuptools
   # curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
   # python get-pip.py
   # pip install docker-compose behave
   ```
   
   For SLES
   ```
   # zypper install python-devel python-setuptools
   # curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
   # python get-pip.py
   # pip install docker-compose behave
   ```

New behave tests require python grpcio package. To install if you haven't already, pick a directory outside the Hyperledger source tree,

   ```
   # git clone https://github.com/gongsu832/grpc.git
   # cd grpc
   # git submodule update --init
   # pip install -rrequirements.txt
   # git checkout tags/release-0_13_1
   # GRPC_PYTHON_BUILD_WITH_CYTHON=1 pip install .
   ```
   
Then run the behave tests (from the fabric directory),

   ```
   # make behave
   ```
   
You should have no failed tests. Note that to rerun the tests, you need to clean up all the leftover containers and images except hyperledger/* images.

If you have tests failing for reason like "cannot connect to Docker endpoint", check the IP address of interface docker0 on your system. In compose-defaults.yml file, the IP address is assumed to be 172.17.0.1. You must change it to match what you have on your system.

If you have tests failing for reason like "failed to GET url http://172.17.0.2:5000/...", it's likely due to the use of http_proxy and/or https_proxy settings in your environment. Please remove these settings before performing the tests.
