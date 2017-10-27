
cd /root
wget https://ftp.pcre.org/pub/pcre/pcre-8.00.tar.gz
tar -zxvf pcre-8.00.tar.gz
cd pcre-8.00/
./configure
make & make install


wget https://www.openssl.org/source/openssl-1.1.0f.tar.gz
tar -zxvf openssl-1.1.0f.tar.gz
cd openssl-1.1.0f/
./config
make & make install


wget http://www.zlib.net/zlib-1.2.11.tar.gz
tar -zxvf zlib-1.2.11.tar.gz
cd zlib-1.2.11/
 ./configure
 make&&make install



wget http://nginx.org/download/nginx-1.13.6.tar.gz
tar -zxvf nginx-1.13.6.tar.gz
cd nginx-1.13.6/
./configure --prefix=/usr/local/nginx 
make && make install
