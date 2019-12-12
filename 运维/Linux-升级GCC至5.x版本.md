# CentOS 6/7 升级GCC/G++至5.x版本  

```console
yum install centos-release-scl-rh
yum -y install devtoolset-4-gcc devtoolset-4-gcc-c++
scl enable devtoolset-4 bash
echo "source /opt/rh/devtoolset-4/enable" >> ~/.bashrc
```



# ubuntu 14.04 升级GCC/G++至5.x版本  

```console
add-apt-repository ppa:ubuntu-toolchain-r/test && apt-get update
apt-get install gcc-5 g++5
 
ln -s /usr/bin/gcc-5 /usr/bin/gcc -f
ln -s /usr/bin/gcc-ar-5 /usr/bin/gcc-ar -f
ln -s /usr/bin/gcc-nm /usr/bin/gcc-nm -f
ln -s /usr/bin/g++-nm /usr/bin/g++-nm -f
ln -s /usr/bin/g++-ar-5 /usr/bin/g++-ar -f
ln -s /usr/bin/g++-5 /usr/bin/g++ -f
```