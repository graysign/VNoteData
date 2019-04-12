###### 安装m4  
```bash
wget http://mirrors.kernel.org/gnu/m4/m4-1.4.13.tar.gz \
&& tar -xzvf m4-1.4.13.tar.gz \
&& cd m4-1.4.13 \
&& ./configure --prefix=/usr/local
make && make install
cd ..
```

###### 安装autoconf  
```bash
wget http://mirrors.kernel.org/gnu/autoconf/autoconf-2.65.tar.gz \
&& tar -xzvf autoconf-2.65.tar.gz \
&& cd autoconf-2.65 \
&& ./configure --prefix=/usr/local
make && make install
cd ..
```

###### 安装automake  
```bash
wget http://mirrors.kernel.org/gnu/automake/automake-1.11.tar.gz \
&& tar xzvf automake-1.11.tar.gz \
&& cd automake-1.11 \
&& ./configure --prefix=/usr/local
make && make install
cd ..
```

###### 安装libtool  
```bash
wget http://mirrors.kernel.org/gnu/libtool/libtool-2.2.6b.tar.gz \
&& tar xzvf libtool-2.2.6b.tar.gz \
&& cd libtool-2.2.6b \
&& ./configure --prefix=/usr/local
make && make install
cd ..
```