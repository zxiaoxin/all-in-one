kolla项目目前更新到3.0.0，而本实验部署的是kolla的tag2.0.3
===
```
虚拟机基本环境：

centos7.2双网卡 
docker版本：1.12.1
ansible：1.9.6 #在2.0.3上ansible的版本不能高于2.0.0，在3.0.0上是允许的
IP：192.168.101.46
主机名：localhost
```

* systemctl stop firewalld && systemctl disable firewalld
*实验环境下不想麻烦直接关闭防火墙了，如果是在服务器上要配置iptables保证安全*

* vim /etc/hosts
    192.168.101.46 localhost<br>
    127.0.0.1 localhost<br>
*安装依赖：*
yum install -y epel-release 
yum install -y python-pip
pip install --upgrade pip
yum install -y python-devel libffi-devel openssl-devel gcc vim git python-setuptools wget
安装ansible1.9.6：
yum install ansible1.9.noarch #在这边ansible的版本不要高于2.0.0，不然在kolla-ansible检测时会报错

[root@localhost ~]# yum install ansible1.9.noarch
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * epel: mirror01.idc.hinet.net
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package ansible1.9.noarch 0:1.9.6-2.el7 will be installed
--> Processing Dependency: sshpass for package: ansible1.9-1.9.6-2.el7.noarch
--> Processing Dependency: python-paramiko for package: ansible1.9-1.9.6-2.el7.noarch
--> Processing Dependency: python-keyczar for package: ansible1.9-1.9.6-2.el7.noarch
--> Processing Dependency: python-jinja2 for package: ansible1.9-1.9.6-2.el7.noarch
--> Processing Dependency: python-httplib2 for package: ansible1.9-1.9.6-2.el7.noarch
--> Processing Dependency: PyYAML for package: ansible1.9-1.9.6-2.el7.noarch
--> Running transaction check
---> Package PyYAML.x86_64 0:3.10-11.el7 will be installed
.......
Installed:
  ansible1.9.noarch 0:1.9.6-2.el7                                                                                                                                      

Dependency Installed:
  PyYAML.x86_64 0:3.10-11.el7                libtomcrypt.x86_64 0:1.17-23.el7         libtommath.x86_64 0:0.42.0-4.el7        libyaml.x86_64 0:0.1.4-11.el7_0        
  python-babel.noarch 0:0.9.6-8.el7          python-httplib2.noarch 0:0.7.7-3.el7     python-jinja2.noarch 0:2.7.2-2.el7      python-keyczar.noarch 0:0.71c-2.el7    
  python-markupsafe.x86_64 0:0.11-10.el7     python-pyasn1.noarch 0:0.1.6-2.el7       python2-crypto.x86_64 0:2.6.1-9.el7     python2-ecdsa.noarch 0:0.13-4.el7      
  python2-paramiko.noarch 0:1.16.1-1.el7     sshpass.x86_64 0:1.05-5.el7             

Complete!
[root@localhost ~]# ansible --version
ansible 1.9.6
  configured module search path = None

安装docker：
curl -sSL https://get.docker.io | bash #版本是1.12.1, build 23cf638 

[root@localhost ~]# curl -sSL https://get.docker.io | bash
+ sh -c 'sleep 3; yum -y -q install docker-engine'
warning: /var/cache/yum/x86_64/7/docker-main-repo/packages/docker-engine-selinux-1.12.1-1.el7.centos.noarch.rpm: Header V4 RSA/SHA512 Signature, key ID 2c52609d: NOKEY
docker-engine-selinux-1.12.1-1.el7.centos.noarch.rpm 的公钥尚未安装
导入 GPG key 0x2C52609D:
 用户ID     : "Docker Release Tool (releasedocker) <docker@docker.com>"
 指纹       : 5811 8e89 f3a9 1289 7c07 0adb f762 2157 2c52 609d
 来自       : https://yum.dockerproject.org/gpg
restorecon:  lstat(/var/lib/docker) failed:  No such file or directory
warning: %post(docker-engine-selinux-1.12.1-1.el7.centos.noarch) scriptlet failed, exit status 255
Non-fatal POSTIN scriptlet failure in rpm package docker-engine-selinux-1.12.1-1.el7.centos.noarch

If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:

  sudo usermod -aG docker your-user

Remember that you will have to log out and back in for this to take effect!

[root@localhost ~]# docker --version
Docker version 1.12.1, build 23cf638

配置docker文件，启动MountFlags选项，默认未开启，未配置会在deploy时 neutron-dhcp-agent容器抛出APIError/HTTPError：
mkdir -p /etc/systemd/system/docker.service.d

tee /etc/systemd/system/docker.service.d/kolla.conf <<-'EOF'
[Service]
MountFlags=shared
EOF

加载配置文件设置自启，并启动docker服务：
systemctl daemon-reload && systemctl enable docker && systemctl start docker

[root@localhost ~]# systemctl daemon-reload && systemctl enable docker && systemctl start docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.

下载kolla代码：
git clone https://git.openstack.org/openstack/kolla -b stable/mitaka

[root@localhost ~]# git clone https://git.openstack.org/openstack/kolla -b stable/mitaka
Cloning into 'kolla'...
remote: Counting objects: 42020, done.
remote: Compressing objects: 100% (22009/22009), done.
remote: Total 42020 (delta 25059), reused 34193 (delta 18256)
Receiving objects: 100% (42020/42020), 5.35 MiB | 277.00 KiB/s, done.
Resolving deltas: 100% (25059/25059), done.

安装kolla：
pip install kolla/ 

[root@localhost ~]# pip install kolla/
Processing ./kolla
Collecting pbr>=1.6 (from kolla==2.0.3.dev34)
  Downloading pbr-1.10.0-py2.py3-none-any.whl (96kB)
    100% |████████████████████████████████| 102kB 98kB/s 
Collecting docker-py>=1.6.0 (from kolla==2.0.3.dev34)
  Downloading docker_py-1.10.3-py2.py3-none-any.whl (48kB)
    100% |████████████████████████████████| 51kB 230kB/s 
Collecting Jinja2>=2.8 (from kolla==2.0.3.dev34)
  Using cached Jinja2-2.8-py2.py3-none-any.whl
Collecting gitdb>=0.6.4 (from kolla==2.0.3.dev34)
  Downloading gitdb-0.6.4.tar.gz (400kB)
    100% |████████████████████████████████| 409kB 364kB/s 
Collecting GitPython>=1.0.1 (from kolla==2.0.3.dev34)
  Downloading GitPython-2.0.8.tar.gz (407kB)
    100% |████████████████████████████████| 409kB 216kB/s 
Requirement already satisfied (use --upgrade to upgrade): six>=1.9.0 in /usr/lib/python2.7/site-packages (from kolla==2.0.3.dev34)
Collecting oslo.config>=3.7.0 (from kolla==2.0.3.dev34)
  Downloading oslo.config-3.17.0-py2.py3-none-any.whl (97kB)
    100% |████████████████████████████████| 102kB 160kB/s 
Collecting graphviz>=0.4.0 (from kolla==2.0.3.dev34)
  Downloading graphviz-0.5.1-py2.py3-none-any.whl
Collecting beautifulsoup4 (from kolla==2.0.3.dev34)
  Downloading beautifulsoup4-4.5.1-py2-none-any.whl (83kB)
    100% |████████████████████████████████| 92kB 111kB/s 
Requirement already satisfied (use --upgrade to upgrade): setuptools!=24.0.0,>=16.0 in /usr/lib/python2.7/site-packages (from kolla==2.0.3.dev34)
Requirement already satisfied (use --upgrade to upgrade): pycrypto>=2.6 in /usr/lib64/python2.7/site-packages (from kolla==2.0.3.dev34)
Collecting requests<2.11,>=2.5.2 (from docker-py>=1.6.0->kolla==2.0.3.dev34)
  Downloading requests-2.10.0-py2.py3-none-any.whl (506kB)
    100% |████████████████████████████████| 512kB 319kB/s 
Collecting backports.ssl-match-hostname>=3.5; python_version < "3.5" (from docker-py>=1.6.0->kolla==2.0.3.dev34)
  Downloading backports.ssl_match_hostname-3.5.0.1.tar.gz
Requirement already satisfied (use --upgrade to upgrade): ipaddress>=1.0.16; python_version < "3.3" in /usr/lib/python2.7/site-packages (from docker-py>=1.6.0->kolla==2.0.3.dev34)
....
  Found existing installation: backports.ssl-match-hostname 3.4.0.2
    Uninstalling backports.ssl-match-hostname-3.4.0.2:
      Successfully uninstalled backports.ssl-match-hostname-3.4.0.2
  Running setup.py install for backports.ssl-match-hostname ... done
  Running setup.py install for websocket-client ... done
  Found existing installation: Jinja2 2.7.2
    Uninstalling Jinja2-2.7.2:
      Successfully uninstalled Jinja2-2.7.2
  Running setup.py install for smmap ... done
  Running setup.py install for gitdb ... done
  Running setup.py install for GitPython ... done
  Found existing installation: Babel 0.9.6
    Uninstalling Babel-0.9.6:
      Successfully uninstalled Babel-0.9.6
  Running setup.py install for wrapt ... done
  Running setup.py install for kolla ... done
Successfully installed Babel-2.3.4 GitPython-2.0.8 Jinja2-2.8 backports.ssl-match-hostname-3.5.0.1 beautifulsoup4-4.5.1 debtcollector-1.8.0 docker-py-1.10.3 docker-pycreds-0.2.1 funcsigs-1.0.2 gitdb-0.6.4 graphviz-0.5.1 kolla-2.0.3.dev34 netaddr-0.7.18 oslo.config-3.17.0 oslo.i18n-3.9.0 pbr-1.10.0 pytz-2016.7 requests-2.10.0 rfc3986-0.4.1 smmap-0.9.0 stevedore-1.17.1 websocket-client-0.37.0 wrapt-1.10.8

进入kolla目录下
cd kolla/

安装tox：
pip install -U tox

安装openstackclient跟neutronclient
pip install -U python-openstackclient python-neutronclient

安装依赖
pip install -r requirements.txt -r test-requirements.txt

[root@localhost kolla]# pip install -r requirements.txt -r test-requirements.txt
Requirement already satisfied (use --upgrade to upgrade): pbr>=1.6 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 4))
Requirement already satisfied (use --upgrade to upgrade): docker-py>=1.6.0 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 5))
Requirement already satisfied (use --upgrade to upgrade): Jinja2>=2.8 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 6))
Requirement already satisfied (use --upgrade to upgrade): gitdb>=0.6.4 in /usr/lib64/python2.7/site-packages (from -r requirements.txt (line 7))
Requirement already satisfied (use --upgrade to upgrade): GitPython>=1.0.1 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 8))
Requirement already satisfied (use --upgrade to upgrade): six>=1.9.0 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 9))
Requirement already satisfied (use --upgrade to upgrade): oslo.config>=3.7.0 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 10))
Requirement already satisfied (use --upgrade to upgrade): graphviz>=0.4.0 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 11))
Requirement already satisfied (use --upgrade to upgrade): beautifulsoup4 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 12))
Requirement already satisfied (use --upgrade to upgrade): setuptools!=24.0.0,>=16.0 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 13))
Requirement already satisfied (use --upgrade to upgrade): pycrypto>=2.6 in /usr/lib64/python2.7/site-packages (from -r requirements.txt (line 14))
Collecting bandit>=0.17.3 (from -r test-requirements.txt (line 4))
  Downloading bandit-1.1.0-py2.py3-none-any.whl (114kB)
.....
  Running setup.py install for mccabe ... done
  Running setup.py install for pyinotify ... done
  Running setup.py install for python-mimeparse ... done
  Running setup.py install for testrepository ... done
  Found existing installation: requests 2.11.1
    Uninstalling requests-2.11.1:
      Successfully uninstalled requests-2.11.1
  Found existing installation: python-keystoneclient 3.5.0
    Uninstalling python-keystoneclient-3.5.0:
      Successfully uninstalled python-keystoneclient-3.5.0
  Running setup.py install for docutils ... done
Successfully installed Pygments-2.1.3 argparse-1.4.0 bandit-1.1.0 bashate-0.5.1 docutils-0.12 extras-1.0.0 fixtures-3.0.0 flake8-2.5.5 futures-3.0.5 hacking-0.11.0 kazoo-2.2.1 linecache2-1.0.0 mccabe-0.2.1 mock-2.0.0 mox3-0.18.0 oslo.context-2.9.0 oslo.log-3.16.0 oslosphinx-4.7.0 oslotest-2.10.0 pep8-1.5.7 pyflakes-0.8.1 pyinotify-0.9.6 python-barbicanclient-4.1.0 python-ceilometerclient-2.6.1 python-dateutil-2.5.3 python-heatclient-1.5.0 python-keystoneclient-2.3.1 python-mimeparse-1.5.3 python-subunit-1.2.0 python-swiftclient-3.1.0 reno-1.8.0 requests-2.10.0 sphinx-1.2.3 testrepository-0.0.20 testscenarios-0.5.0 testtools-2.2.0 traceback2-1.4.0 unittest2-1.1.0 zake-0.2.2

生成kolla-build.conf配置文件：
tox -e genconfig

[root@localhost kolla]# tox -e genconfig
genconfig create: /root/kolla/.tox/genconfig
genconfig installdeps: -r/root/kolla/requirements.txt, -r/root/kolla/test-requirements.txt
genconfig develop-inst: /root/kolla
genconfig installed: appdirs==1.4.0,Babel==2.3.4,backports.ssl-match-hostname==3.5.0.1,bandit==1.1.0,bashate==0.5.1,beautifulsoup4==4.5.1,cliff==2.2.0,cmd2==0.6.9,debtcollector==1.8.0,docker-py==1.10.3,docker-pycreds==0.2.1,docutils==0.12,extras==1.0.0,fixtures==3.0.0,flake8==2.5.5,funcsigs==1.0.2,functools32==3.2.3.post2,futures==3.0.5,gitdb==0.6.4,GitPython==2.0.8,graphviz==0.5.1,hacking==0.11.0,ipaddress==1.0.17,iso8601==0.1.11,Jinja2==2.8,jsonpatch==1.14,jsonpointer==1.10,jsonschema==2.5.1,kazoo==2.2.1,keystoneauth1==2.12.1,-e git+https://git.openstack.org/openstack/kolla@dee79bcf897f3a443f88718788f91503cb538941#egg=kolla,linecache2==1.0.0,MarkupSafe==0.23,mccabe==0.2.1,mock==2.0.0,monotonic==1.2,mox3==0.18.0,msgpack-python==0.4.8,netaddr==0.7.18,netifaces==0.10.5,os-client-config==1.21.1,osc-lib==1.1.0,oslo.config==3.17.0,oslo.context==2.9.0,oslo.i18n==3.9.0,oslo.log==3.16.0,oslo.serialization==2.13.0,oslo.utils==3.16.0,oslosphinx==4.7.0,oslotest==2.10.0,pbr==1.10.0,pep8==1.5.7,positional==1.1.1,prettytable==0.7.2,pycrypto==2.6.1,pyflakes==0.8.1,Pygments==2.1.3,pyinotify==0.9.6,pyparsing==2.1.9,python-barbicanclient==4.1.0,python-ceilometerclient==2.6.1,python-cinderclient==1.9.0,python-dateutil==2.5.3,python-glanceclient==2.5.0,python-heatclient==1.5.0,python-keystoneclient==2.3.1,python-mimeparse==1.5.3,python-neutronclient==6.0.0,python-novaclient==6.0.0,python-subunit==1.2.0,python-swiftclient==3.1.0,pytz==2016.7,PyYAML==3.12,reno==1.8.0,requests==2.10.0,requestsexceptions==1.1.3,rfc3986==0.4.1,simplejson==3.8.2,six==1.10.0,smmap==0.9.0,Sphinx==1.2.3,stevedore==1.17.1,testrepository==0.0.20,testscenarios==0.5.0,testtools==2.2.0,traceback2==1.4.0,unicodecsv==0.14.1,unittest2==1.1.0,warlock==1.2.0,websocket-client==0.37.0,wrapt==1.10.8,zake==0.2.2
genconfig runtests: PYTHONHASHSEED='3383939239'
genconfig runtests: commands[0] | oslo-config-generator --config-file etc/oslo-config-generator/kolla-build.conf
WARNING:stevedore.named:Could not load kolla
_______________________________________________________________________________ summary _______________________________________________________________________________
  genconfig: commands succeeded
  congratulations :)

复制配置文件到/etc目录下
cp -rv etc/kolla /etc/

[root@localhost kolla]# cp -rv etc/kolla /etc/
‘etc/kolla’ -> ‘/etc/kolla’
‘etc/kolla/passwords.yml’ -> ‘/etc/kolla/passwords.yml’
‘etc/kolla/globals.yml’ -> ‘/etc/kolla/globals.yml’
‘etc/kolla/kolla-build.conf’ -> ‘/etc/kolla/kolla-build.conf’

kolla-build --base centos --type source -p default
#这边选择build基础的镜像，如果业务上需求，去掉-p default就build全部
#如果出现build错误，可以单独build，单独build步骤如下：（成功就不要这样做啦）
vim /etc/kolla/kolla-build.conf 
#install_type = binary #注释掉这个改成source
install_type = source

#配置中默认是centos的，所以没有改base，如果是ubuntu请自行更改
#保存退出再build就可以build source镜像
#镜像下来，就完成一大步啦

vim /etc/kolla/passwords.yml
kolla_install_type: "source"
kolla_internal_address: "192.168.101.147" #这个IP要跟端口同一个网段并且未被使用
network_interface: "enp4s0.2"
neutron_external_interface: "enp4s0.3"

kolla-genpwd
#生成密码，也可以自己去配置

kolla-ansible prechecks
#检查端口，IP，ansible等配置

kolla-ansible deploy #开始部署容器
#没有错误就是成功啦

kolla-ansible post-deploy

cat /etc/kolla/admin-openrc.sh 
#查看dashboard的登录信息

source /etc/kolla/admin-openrc.sh
#加载环境变量

kolla/tools/init-runonce
#初始化一个镜像跟网络

