sish
====

An open source serveo/ngrok alternative.

Deploy
------

Builds are made automatically on Google Cloud Build and Dockerhub. Feel free to either use the automated binaries or to build your own. If you submit a PR and would like access to Google Cloud Build's output (including pre-made PR binaries), feel free to let me know.

1. Pull the Docker image
    - `docker pull antoniomika/sish:latest`
2. Run the image

    - ```bash
      docker run -itd --name sish \
        -v ~/sish/ssl:/ssl \
        -v ~/sish/keys:/keys \
        -v ~/sish/pubkeys:/pubkeys \
        --net=host antoniomika/sish:latest \
        -sish.addr=:22 \
        -sish.https=:443 \
        -sish.http=:80 \
        -sish.httpsenabled=true \
        -sish.httpspems=/ssl \
        -sish.keysdir=/pubkeys \
        -sish.pkloc=/keys/ssh_key \
        -sish.bindrandom=false
      ```

3. SSH to your host to communicate with sish
    - `ssh -p 2222 -R 80:localhost:8080 ssi.sh`

Docker Compose
--------------

You can also use Docker Compose to setup your sish instance. This includes taking care of SSL via Let's Encrypt for you. This uses the [adferrand/docker-letsencrypt-dns](https://github.com/adferrand/docker-letsencrypt-dns) container to handle issuing wildcard certifications over DNS. For more information on how to use this, head to that link above. Generally, you can deploy your service like so:

```bash
DOMAIN=yourdomain.com \
LETSENCRYPT_USER_MAIL=you@yourdomain.com \
LEXICON_PROVIDER=cloudflare \
LEXICON_PROVIDER_OPTIONS="--auth-username=you@yourdomain.com --auth-token=your-auth-token" \
docker-compose -f deploy/docker-compose.yml up -d
```

How it works
------------

SSH can normally forward local and remote ports. This service implements an SSH server that only does that and nothing else. The service supports multiplexing connections over HTTP/HTTPS with WebSocket support. Just assign a remote port as port `80` to proxy HTTP traffic and `443` to proxy HTTPS traffic. If you use any other remote port, the server will listen to the port for connections, but only if that port is available.

You can choose your own subdomain instead of relying on a randomly assigned one
by setting the `-sish.forcerandomsubdomain` option to `false` and then selecting a
subdomain by prepending it to the remote port specifier:

`ssh -p 2222 -R foo:80:localhost:8080 ssi.sh`

If the selected subdomain is not taken, it will be assigned to your connection.

Authentication
--------------

If you want to use this service privately, it supports both public key and password authentication. To enable authentication, set `-sish.auth=true` as one of your CLI options and be sure to configure `-sish.password` or `-sish.keysdir` to your liking. The directory provided by `-sish.keysdir` is watched for changes and will reload the authorized keys automatically. The authorized cert index is regenerated on directory modification, so removed public keys will also automatically be removed. Files in this directory can either be single key per file, or multiple keys per file separated by newlines, similar to `authorized_keys`. Password auth can be disabled by setting `-sish.password=""` as a CLI option.

One of my favorite ways of using this for authentication is like so:

```bash
sish@sish0:~/sish/pubkeys# curl https://github.com/antoniomika.keys > antoniomika
```

This will load my public keys from GitHub, place them in the directory that sish is watching, and then load the pubkey. As soon as this command is run, I can SSH normally and it will authorize me.

Whitelisting IPs
----------------

Whitelisting IP ranges or countries is also possible. Whole CIDR ranges can be
specified with the `-sish.whitelistedips` option that accepts a comma-separated string like "192.30.252.0/22,185.199.108.0/22". If you want to whitelist a single
IP, use the `/32` range.

To whitelist countries, use `sish.whitelistedcountries` with a comma-separated
string of countries in ISO format (for example, "pt" for Portugal). You'll also
need to set `-sish.usegeodb` to `true`.

Demo - At this time, the demo instance has been set to require auth due to abuse
----

There is a demo service (and my private instance) currently running on `ssi.sh` that doesn't require any authentication. This service provides default logging (errors, connection IP/username, and pubkey fingerprint). I do not log any of the password authentication data or the data sent within the service/tunnels. My deploy uses the exact deploy steps that are listed above. This instance is for testing and educational purposes only. You can deploy this extremely easily on any host (Google Cloud Platform provides an always-free instance that this should run perfectly on). If the service begins to accrue a lot of traffic, I will enable authentication and then you can reach out to me to get your SSH key whitelisted (make sure it's on GitHub and you provide me with your GitHub username).

Notes
-----

1. This is by no means production ready in any way. This was hacked together and solves a fairly specific use case.
      - You can help it get production ready by submitting PRs/reviewing code/writing tests/etc
2. This is a fairly simple implementation, I've intentionally cut corners in some places to make it easier to write.
3. If you have any questions or comments, feel free to reach out via email [me@antoniomika.me](mailto:me@antoniomika.me) or on [freenode IRC #sish](https://kiwiirc.com/client/chat.freenode.net:6697/#sish)

CLI Flags
---------

```text
sh-3.2# ./sish -h
Usage of ./sish:
  -sish.addr string
        The address to listen for SSH connections (default "localhost:2222")
  -sish.auth
        Whether or not to require auth on the SSH service
  -sish.bannedcountries string
        A comma separated list of banned countries
  -sish.bannedips string
        A comma separated list of banned ips
  -sish.bannedsubdomains string
        A comma separated list of banned subdomains (default "localhost")
  -sish.bindrandom
        Bind ports randomly (OS chooses) (default true)
  -sish.bindrange string
        Ports that are allowed to be bound (default "0,1024-65535")
  -sish.cleanupunbound
        Whether or not to cleanup unbound (forwarded) SSH connections (default true)
  -sish.debug
        Whether or not to print debug information
  -sish.domain string
        The domain for HTTP(S) multiplexing (default "ssi.sh")
  -sish.forcerandomsubdomain
        Whether or not to force a random subdomain (default true)
  -sish.http string
        The address to listen for HTTP connections (default "localhost:80")
  -sish.httpport int
        The port to use for http command output
  -sish.https string
        The address to listen for HTTPS connections (default "localhost:443")
  -sish.httpsenabled
        Whether or not to listen for HTTPS connections
  -sish.httpspems string
        The location of pem files for HTTPS (fullchain.pem and privkey.pem) (default "ssl/")
  -sish.httpsport int
        The port to use for https command output
  -sish.keysdir string
        Directory for public keys for pubkey auth (default "pubkeys/")
  -sish.logtoclient
        Whether or not to log http requests to the client
  -sish.password string
        Password to use for password auth (default "S3Cr3tP4$$W0rD")
  -sish.pkloc string
        SSH server private key (default "keys/ssh_key")
  -sish.pkpass string
        Passphrase to use for the server private key (default "S3Cr3tP4$$phrAsE")
  -sish.proxyprotoenabled
        Whether or not to enable the use of the proxy protocol
  -sish.proxyprotoversion string
        What version of the proxy protocol to use. Can either be 1, 2, or userdefined. If userdefined, the user needs to add a command to SSH called proxyproto:version (ie proxyproto:1) (default "1")
  -sish.redirectroot
        Whether or not to redirect the root domain (default true)
  -sish.redirectrootlocation string
        Where to redirect the root domain to (default "https://github.com/antoniomika/sish")
  -sish.subdomainlen int
        The length of the random subdomain to generate (default 3)
  -sish.tcpalias
        Whether or not to allow the use of TCP aliasing
  -sish.usegeodb
        Whether or not to use the maxmind geodb
  -sish.verifyorigin
        Whether or not to verify origin on websocket connection (default true)
  -sish.verifyssl
        Whether or not to verify SSL on proxy connection (default true)
  -sish.version
        Print version and exit
  -sish.whitelistedcountries string
        A comma separated list of whitelisted countries
  -sish.whitelistedips string
        A comma separated list of whitelisted ips
```

说明：sish是一个SSH服务器，仅用于远程端口转发，可以快速将本地端口暴露在外网，作者声称其为Servo/Ngrok替代方案，仅使用SSH的HTTP(S)、WS(S)、TCP隧道连接到他们的localhost服务器，该工具和Servo差不多一样，不同就是Servo官方提供了免费的SSH客户端，而sish作者提供的客户端貌似因为滥用关闭了，所以就需要我们自己搭建了，这里就水下Docker和手动安装。

Docker安装
Github地址：https://github.com/antoniomika/sish

1、安装Docker

#CentOS 6
rpm -iUvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum update -y
yum -y install docker-io
service docker start
chkconfig docker on

#CentOS 7、Debian、Ubuntu
curl -sSL https://get.docker.com/ | sh
systemctl start docker
systemctl enable docker
2、拉取镜像
这里由于直接使用ip的话，只能用于转发TCP，HTTP(S)等就需要配置下域名了，所以以下全部默认使用域名。

先解析一个主/泛域名到服务器ip，比如解析moerats.com、*.moerats.com到服务器ip。

然后再参考下面的参数详解，再自行修改部分参数后，使用命令：

#配置http域名
docker run -d --name sish \
  --restart=always \
  -v ~/sish/keys:/keys \
  -v ~/sish/pubkeys:/pubkeys \
  --net=host antoniomika/sish \
  -sish.addr=:3333 \
  -sish.http=:80 \
  -sish.keysdir=/pubkeys \
  -sish.pkloc=/keys/ssh_key \
  -sish.forcerandomsubdomain=false \
  -sish.domain moerats.com \
  -sish.bindrandom=false \
  -sish.redirectrootlocation https://www.baidu.com 

#配置https域名，这里需要提供泛域名证书
docker run -d --name sish \
  --restart=always \
  -v ~/sish/ssl:/ssl \
  -v ~/sish/keys:/keys \
  -v ~/sish/pubkeys:/pubkeys \
  --net=host antoniomika/sish \
  -sish.addr=:3333 \
  -sish.https=:443 \
  -sish.http=:80 \
  -sish.httpsenabled=true \
  -sish.httpspems=/ssl \
  -sish.keysdir=/pubkeys \
  -sish.pkloc=/keys/ssh_key \
  -sish.forcerandomsubdomain=false \
  -sish.domain moerats.com \
  -sish.bindrandom=false \
  -sish.redirectrootlocation https://www.baidu.com
部分参数如下：

-sish.addr=:3333  #ssh监听地址
-sish.forcerandomsubdomain=false  #是否强制随机子域，这个建议关掉
-sish.bindrandom=false  #是否随机绑定端口，这个建议关掉
-sish.domain moerats.com  #使用的域名
-sish.redirectrootlocation https://www.baidu.com  #主域名(-sish.domain参数)强制跳转到该地址
-sish.httpspems=/ssl  #泛域名SSL证书路径，存放路径~/sish/ssl，证书命名格式fullchain.pem和privkey.pem
其他参数默认即可，也可以自行添加或修改其它参数。

全部参数如下：

Usage of sish:
  -sish.addr string
        The address to listen for SSH connections (default "localhost:2222")
  -sish.auth
        Whether or not to require auth on the SSH service
  -sish.bannedcountries string
        A comma separated list of banned countries
  -sish.bannedips string
        A comma separated list of banned ips
  -sish.bannedsubdomains string
        A comma separated list of banned subdomains (default "localhost")
  -sish.bindrandom
        Bind ports randomly (OS chooses) (default true)
  -sish.bindrange string
        Ports that are allowed to be bound (default "0,1024-65535")
  -sish.cleanupunbound
        Whether or not to cleanup unbound (forwarded) SSH connections (default true)
  -sish.debug
        Whether or not to print debug information
  -sish.domain string
        The domain for HTTP(S) multiplexing (default "ssi.sh")
  -sish.forcerandomsubdomain
        Whether or not to force a random subdomain (default true)
  -sish.http string
        The address to listen for HTTP connections (default "localhost:80")
  -sish.httpport int
        The port for HTTP connections. This is only for output messages (default 80)
  -sish.https string
        The address to listen for HTTPS connections (default "localhost:443")
  -sish.httpsenabled
        Whether or not to listen for HTTPS connections
  -sish.httpspems string
        The location of pem files for HTTPS (fullchain.pem and privkey.pem) (default "ssl/")
  -sish.httpsport int
        The port for HTTPS connections. This is only for output messages (default 443)
  -sish.keysdir string
        Directory for public keys for pubkey auth (default "pubkeys/")
  -sish.password string
        Password to use for password auth (default "S3Cr3tP4$$W0rD")
  -sish.pkloc string
        SSH server private key (default "keys/ssh_key")
  -sish.pkpass string
        Passphrase to use for the server private key (default "S3Cr3tP4$$phrAsE")
  -sish.proxyprotoenabled
        Whether or not to enable the use of the proxy protocol
  -sish.proxyprotoversion string
        What version of the proxy protocol to use. Can either be 1, 2, or userdefined. If userdefined, the user needs to add a command to SSH called proxy:version (ie proxy:1) (default "1")
  -sish.redirectroot
        Whether or not to redirect the root domain (default true)
  -sish.redirectrootlocation string
        Where to redirect the root domain to (default "https://github.com/antoniomika/sish")
  -sish.subdomainlen int
        The length of the random subdomain to generate (default 3)
  -sish.usegeodb
        Whether or not to use the maxmind geodb
  -sish.verifyorigin
        Whether or not to verify origin on websocket connection (default true)
  -sish.verifyssl
        Whether or not to verify SSL on proxy connection (default true)
  -sish.whitelistedcountries string
        A comma separated list of whitelisted countries
  -sish.whitelistedips string
        A comma separated list of whitelisted ips
看不懂的，可以使用下谷歌翻译。

最后CentOS系统建议关闭防火墙使用，或者打开部分端口也行，关闭命令：

#CentOS 6系统
service iptables stop
chkconfig iptables off

#CentOS 7系统
systemctl stop firewalld
systemctl disable firewalld
像阿里云等服务器，还需要去安全组那里开放下端口。

手动安装
Docker虽然方便很多，但也有人会喜欢手动安装，这里作者没直接给出二进制文件，所以就需要我们手动来构建二进制文件了。

1、安装Go
这里由于需要新版的Go环境，所以这里就使用Go二进制包安装环境，下载地址→传送门。

然后根据自己的服务器架构下载对应的最新安装包，一般可以直接使用命令：

#32位系统下载
wget -O go.tar.gz https://dl.google.com/go/go1.13.3.linux-386.tar.gz
#64位系统下载
wget -O go.tar.gz https://dl.google.com/go/go1.13.3.linux-amd64.tar.gz

#解压压缩包
tar -zxvf go.tar.gz -C /usr/local
#设置环境变量，将以下一起复制进ssh客户端运行
mkdir $HOME/go
echo 'export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin' >> /etc/profile
source /etc/profile
#查看go版本，有输出即为安装成功
go version
2、安装sish

#下载源码到主目录
git clone https://github.com/antoniomika/sish
cd sish
#编译二进制文件
go install
这里提示-bash: git: command not found的，可以先使用命令：

#CentOS
yum -y install git

#Debian、Ubuntu
apt install git -y
3、运行sish
运行参数这里就不贴了，直接参考上面Docker安装最下面的全部参数就行了。

先解析一个主/泛域名到服务器ip，比如解析moerats.com、*.moerats.com到服务器ip。

这里就贴个大概需要使用的参数，其它的根据需求自行修改，使用命令：

#配置http域名
sish -sish.addr=:3333 -sish.http=:80 -sish.domain moerats.com -sish.forcerandomsubdomain=false -sish.bindrandom=false -sish.redirectrootlocation https://www.moerats.com -sish.keysdir=/sish/pubkeys -sish.pkloc=/sish/keys/ssh_key 

#配置https域名
sish -sish.addr=:3333 -sish.https=:443 -sish.http=:80 -sish.domain moerats.com -sish.forcerandomsubdomain=false -sish.bindrandom=false -sish.httpsenabled=true -sish.redirectrootlocation https://www.moerats.com -sish.keysdir=/sish/pubkeys -sish.pkloc=/sish/keys/ssh_key -sish.httpspems=/sish/ssl
部分参数详解：

-sish.addr=:3333  #ssh监听地址，这里为3333
-sish.forcerandomsubdomain=false  #是否强制随机子域，这个建议关掉
-sish.bindrandom=false  #是否随机绑定端口，这个建议关掉
-sish.domain moerats.com  #使用的域名
-sish.redirectrootlocation https://www.baidu.com  #主域名(-sish.domain参数)强制跳转到该地址
-sish.httpspems=/sish/ssl  #泛域名SSL证书存放路径，证书命名格式fullchain.pem和privkey.pem
-sish.keysdir=/sish/pubkeys  #pubkey auth的公共密钥存放文件夹
-sish.pkloc=/sish/keys/ssh_key  #SSH服务器私钥
这里/sish/ssl、/sish/pubkeys、/sish/keys目录需要自己提前创建下，使用命令：

mkdir -p /sish/ssl /sish/pubkeys /sish/keys
4、开机自启
如果你使用手动命令没问题了，先使用Ctrl+C断开命令。

再新建systemd配置文件，适用CentOS 7、Debian 8+、Ubuntu 16+。

#修改成你手动运行命令的全部参数
command="-sish.addr=:3333 -sish.http=:80 -sish.domain moerats.com -sish.forcerandomsubdomain=false -sish.bindrandom=false -sish.redirectrootlocation https://www.moerats.com -sish.keysdir=/sish/pubkeys -sish.pkloc=/sish/keys"
#将以下代码一起复制到SSH运行
cat > /etc/systemd/system/sish.service <<EOF
[Unit]
Description=sish
After=network.target

[Service]
Type=simple
ExecStart=$(command -v sish) ${command}
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
启动并设置开机自启：

systemctl start sish
systemctl enable sish
最后CentOS系统建议关闭防火墙使用，或者打开部分端口也行，关闭命令：

#CentOS 6系统
service iptables stop
chkconfig iptables off

#CentOS 7系统
systemctl stop firewalld
systemctl disable firewalld
像阿里云等服务器，还需要去安全组那里开放下端口。

使用
使用要求：可以使用SSH，并且能连接到互联网，Linux、Windows等系统都行。

以下所使用的的moerats.com为上面配置好的客户端域名地址，自行修改成自己的即可。

1、转发HTTP(S)
将本地3000端口穿透到公网中，使用命令：

#要转发其它端口的自行替换
ssh -p 3333 -R 80:localhost:3000 moerats.com
第一次如果有提示，选择yes即可，之后会为你随机生成一个moerats.com的二级域名，然后就可以使用浏览器间接访问本地的localhost:3000了。

如果要指定二级域名，可以使用命令：

#这里默认为no1.moerats.com，自行替换即可
ssh -p 3333 -R no1:80:localhost:3000 moerats.com
此时你就可以在外网使用no1.moerats.com访问你本地的localhost:3000了。

2、转发TCP
将本地6789端口穿透到公网的9876端口中，使用命令：

#可以自行设置公网端口，这里默认6789，如果你要转发SSH端口，那就改成你的SSH端口
ssh -p 3333 -R 9876:localhost:6789 moerats.com
这里只说了下简单用法，客户端我们还可以设置国家/地区、IP白名单等，使用参考→传送门。

最后没有泛域名证书的，可以查看该教程自己申请→传送门，或者等博主发码子→传送门。
