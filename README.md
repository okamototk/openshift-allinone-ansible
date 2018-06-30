# 下準備 

下記のAnsibleを実行し、openshift-ansibleを実行する下準備を行う。

    # git clone https://github.com/okamototk/openshift-allinone-ansible
    # cd openshift-allinone-ansible

hosts.inventoryを参考にインベントリファイルを作成する。

Ansibleを実行する。

    # ansible-playbook -i hosts site.yml

# サーバの設定
構築した各サーバで下記の設定を行う。

まず、ホストファイルを修正し、ホスト間でホスト名で通信できるようにしておく。

### /etc/hosts

    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    10.0.0.7 os-master1
    10.0.0.8 os-node1


### ボリュームの設定(devicemapperドライバを利用する場合)

/etc/sysconfig/docker-storage-setupに下記の行を追加する。

    VG=docker-vg

LVMのボリュームグループを作成。/dev/sdb1を利用する場合。
予め、Dockerボリューム用のストレージは用意しておく。

    # pvcreate /dev/sdb1
    # vgcreate docker-vg /dev/sdb1

Dockerの初期化、ボリュームグループを設定する

    # systemctl stop docker.service
    # rm -fr /var/lib/docker/*
    # docker-storage-setup
　　# systemctl start docker

# OpenShift Ansibleの実行

openshift-ansibleを取得する

    # git clone https://github.com/openshift/openshift-ansible


OpenShiftの構成を定義したhosts.inventoryを作成する。OpenShiftオリジナルの設定に比べ、内部DNSを作成する[dns]ホストグループが追加されているので注意。

### hosts.inventory

```
[masters]
ose3-master1

[etcd]
os-master1

[nodes]
os-master1 openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
ose3-node[1:2].test.example.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"

[nfs]
ose3-master1.test.example.com

[lb]
ose3-lb.test.example.com

[dns]
ose3-dns1.test.example.com

[OSEv3:children]
masters
nodes
etcd
lb
nfs

[OSEv3:vars]
# Ansibleを利用するユーザ
ansible_user=okamototk

# ansible_userにroot以外を設定した場合、設定
ansible_become=yes

# OpenShiftのバージョンを指定
openshift_deployment_type=origin
openshift_release=v3.9

# アプリケーション・コンテナのドメイン
openshift_master_default_subdomain=apps.test.example.com

# Webコンソールのホスト名
openshift_master_cluster_public_hostname=openshift-ansible.public.example.com

# htpasswdでユーザを管理
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]

# 小さいサイズのVMなどで試す場合は、diskとmeomryチェックを無視
openshift_disable_check=disk_availability,memory_availability

# iptablesの代わりにfirewalldを利用
os_firewall_use_firewalld=True

# VXLANによりプロジェクト間のネットワークを隔離
openshift_use_openshift_sdn=True

# サービスカタログのインストールの無効化
openshift_enable_service_catalog=false

# デバッグレベル
debug_level=2
```

下記のコマンドでansibleを実行

### OpenShiftの事前条件のチェックとパッケージインストール

    # ansible-playbook -i inventory/hosts.myinventory playbooks/prerequisites.yml

### OpenShiftのインストール

    # ansible-playbook -i inventory/hosts.myinventory playbooks/deploy_cluster.yml 


# 確認

OpenShiftにアクセスするWebブラウザのhostsファイルにose3-lbホストにアクセスできるIPアドレスをopenshift-ansible.public.example.comに設定する。Windowsの場合、下記のファイルに記述する。

C:\Windows\System32\drivers\etc\hosts

    ...
    23.96.13.5 openshift-ansible.public.example.com


https://openshift-ansible.public.example.com:8443にブラウザでアクセスする。ユーザadmin、パスワードsmartvmでログインし、アクセスできることを確認する。


## サービスカタログのインストール(オプション)

下記のコマンドでansibleを実行

    $ ansible-playbook -i inventory/hosts.myinventory playbooks/openshift-service-catalog/config.yml

### Loadbalancerへのルータの設定

デフォルトでインストールされるロードバランサはWebConsole/APIのアクセスのみ負荷分散されるように設定されている。
ロードバランサを利用してDNS名でOpenShiftのアプリケーションにアクセスできるように、ロードバランサノード(ose3-lb)の/etc/haproxy/haproxy.cfgに下記の設定を追加する。
(masterノードとinfraノードを兼用した設定の場合。infraノードを別で作成した場合はmaster0/master0のIPアドレスをインフラノードにすること)

#### /etc/haproxy/haproxy.cfg

frontend  openshift-router-http
    bind *:80
    default_backend openshift-router-http
    mode tcp
    option tcplog
    
backend openshift-router-http
    balance source
    mode tcp
    server      master0 10.0.0.5:80 check
    server      master1 10.0.0.6:80 check

frontend  openshift-router-https
    bind *:443
    default_backend openshift-router-https
    mode tcp
    option tcplog
    
backend openshift-router-https
    balance source
    mode tcp
    server      master0 10.0.0.5:443 check
    server      master1 10.0.0.6:443 check

# 動作確認

### CLIの確認

masterノードにログインし、下記のコマンドを実行する。

```
$ ssh ose3-master1
$ oc get all

[okamototk@ose3-master1 ~]$ oc get all
NAME                                      REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfigs/docker-registry         1          1         1         config
deploymentconfigs/registry-console        1          1         1         config
deploymentconfigs/router                  1          1         1         config

NAME                                    DOCKER REPO                                                         TAGS      UPDATED
imagestreams/registry-console           docker-registry.default.svc:5000/default/registry-console           latest    About an hour ago

NAME                           HOST/PORT                                             PATH      SERVICES                PORT       TERMINATION   WILDCARD
routes/docker-registry         docker-registry-default.apps.test.example.com                   docker-registry         <all>      passthrough   None
routes/registry-console        registry-console-default.apps.test.example.com                  registry-console        <all>      passthrough   None

NAME                               READY     STATUS      RESTARTS   AGE
po/docker-registry-1-bfrh6         1/1       Running     0          1h
po/registry-console-1-pbtzd        1/1       Running     0          1h
po/router-1-7k79v                  1/1       Running     0          1h

NAME                         DESIRED   CURRENT   READY     AGE
rc/docker-registry-1         1         1         1         1h
rc/registry-console-1        1         1         1         1h
rc/router-1                  1         1         1         1h

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
svc/docker-registry         ClusterIP   172.30.220.47    <none>        5000/TCP                  1h
svc/kubernetes              ClusterIP   172.30.0.1       <none>        443/TCP,53/UDP,53/TCP     1h
svc/registry-console        ClusterIP   172.30.59.107    <none>        9000/TCP                  1h
svc/router                  ClusterIP   172.30.186.26    <none>        80/TCP,443/TCP,1936/TCP   1h
```

failedなどが表示していると何かおかしい。ので、注意

## GUIの確認

hostsファイル(C:\Windows\System32\drivers\etc\hosts)を記載する。

IPアドレスには、端末からlb(ose3-lb)にアクセスできるIPアドレスを指定し、Web Console、Registory Console, Docker Registoryにアクセスできるようにする。
後で、Springのアプリへのアクセスなど試してみたければ、SpringのアプリのURLを追加する。
Web ConsoleのDNS名はAnsibleのインベントリのopenshift_master_cluster_public_hostnameで指定した値を指定する。

```
23.96.15.7 openshift-ansible.public.example.com registry-console-default.apps.test.example.com  docker-registry-default.apps.test.example.com springboot-sample-app-default.apps.test.example.com 
```

下記のURLにブラウザからアクセスして画面が表示されることを確認する。Docker RegistryはWeb画面ではないので、空白のページが見れればよい。

|---------------------------------------------------------|------------------------|
| URL                                                     |説明                    |
|---------------------------------------------------------|------------------------|
| https://openshift-ansible.public.example.com:8443/      | OpenShift管理コンソール|
|---------------------------------------------------------|------------------------|
| https://registry-console-default.apps.test.example.com/ | Registryコンソール     |
|---------------------------------------------------------|------------------------|
| https://docker-registry-default.apps.test.example.com// | Docker Registry        |
|---------------------------------------------------------|------------------------|

# 使い方

masterノードにSSHでログインし、下記のコマンドでsysadminユーザ(パスワードadmin)を作成する。

    # htpasswd -b /etc/openshift/openshift-passwd  sysadmin admin

adminユーザでログインする。

    # oc login
    Authentication required for https://os-master1.local:8443 (openshift)
    Username: sysadmin
    Password: 
    Login successful.

    You don't have any projects. You can try to create a new project, by running

        oc new-project <projectname>

プロジェクトを作成する。

    # oc new-project test
    Now using project "test" on server "https://os-master1.local:8443".
    
    You can add applications to this project with the 'new-app' command. For example, try:
    
        oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.

SpringBootのサンプルアプリケーションを作成する。

    # oc new-app codecentric/springboot-maven3-centos~https://github.com/codecentric/springboot-sample-app.git

ビルドが完了したかどうか、下記のコマンドで確認する。

    # oc logs -f bc/springboot-sample-app

他のホストからアクセスできるように、Rourterに追加する。

    # oc expose service springboot-sample-app --hostname=springboot-sample-app.your-openshift-installation.com

ルータ経由でアクセスの動作確認を行う

    # curl http://springboot-sample-app.your-openshift-installation.com

エラー画面が表示されなければ、正しく動作している。


     
# トラブルシューティング


## masterノードが正しく動作しない

インストール途中、下記のメッセージが出て途中で止まることがある。

    TASK [openshift_hosted : Ensure OpenShift router correctly rolls out (best-effort today)] ...

上記のようなメッセージが出てrouterやregistryの作成がうまくいかない場合は、スケジューラが正しく
設定されていない可能性がある。その場合は、下記のようにして、スケジューラを有効にする。

```
[root@os-master1 ~]# oc get nodes
NAME                STATUS                     AGE       VERSION
os-master1   Ready,SchedulingDisabled   14m       v1.6.1+5115d708d7
os-node1     Ready                      14m       v1.6.1+5115d708d7
[root@os-master1 ~]# oadm manage-node os-master1.local --schedulable=true
NAME                STATUS    AGE       VERSION
os-master1   Ready     15m       v1.6.1+5115d708d7
[root@os-master1 ~]# oc get nodes
NAME                STATUS    AGE       VERSION
os-master1   Ready     15m       v1.6.1+5115d708d7
os-node1     Ready     15m       v1.6.1+5115d708d7
```
## Docker Resgitryへのpushに失敗する(1)

Docker Resgistryへのpushに下記のメッセージと共に失敗する

    error: build error: Failed to push image: Get https://docker-registry.default.svc:5000/v1/_ping: dial tcp: lookup docker-registry.default.svc on 168.63.129.16:53: no such host


masterノード上でpingを実行し、ローカルドメインとインターネットの名前解決ができていることを確認する。

    # ping docker-registry.default.svc
    PING docker-registry.default.svc.cluster.local (172.30.93.136) 56(84) bytes of data.

    # ping registry.access.redhat.com
    PING registry.access.redhat.com (209.132.182.63) 56(84) bytes of data.
    64 bytes from registry.redhat.io.182.132.209.in-addr.arpa (209.132.182.63): icmp_seq=1 ttl=235 time=136 ms

名前解決ができていなければ、

/etc/resolve.confに自ホストのIPアドレスをネームサーバに指定し、 cluster.localをDNSの検索対象に入っていること

    search cluster.local
    nameserver 192.168.1.151

/etc/origin/node/node-dnsmasq.confに何も設定されていいないこと

/etc/dnsmasq.d/node-dnsmasq.confに下記の設定がされていること

    server=/in-addr.arpa/127.0.0.1
    server=/cluster.local/127.0.0.1

/etc/dnsmasq.d/origin-upstream-dns.confに内部DNSサーバのアドレスが設定されていること。

server=192.168.1.192

を確認する。



## Docker Registryへのpushに失敗する(2)

下記のようなログ表示とともに、イメージのプッシュに失敗し、ｍ

    # oc logs -f bc/springboot-sample-app
    ...
    Pushing image docker-registry.default.svc.cluster.local:5000/test/springboot-sample-app:latest ...
    Warning: Push failed, retrying in 5s ...
    Warning: Push failed, retrying in 5s ...
    Warning: Push failed, retrying in 5s ...
    Warning: Push failed, retrying in 5s ...
    Warning: Push failed, retrying in 5s ...
    Warning: Push failed, retrying in 5s ...
    Warning: Push failed, retrying in 5s ...
    error: build error: Failed to push image: After retrying 6 times, Push image still failed

次のように、Docker Registryがペンディングになっている場合、

    # oc status -v
    In project default on server https://os-master1.local:8443
    
    https://docker-registry-default.router.default.svc.cluster.local (passthrough) (svc/docker-registry)
      dc/docker-registry deploys docker.io/openshift/origin-docker-registry:v3.6.0
        deployment #1 pending 3 hours ago

マスターノードがReadyになっていることを確認する。

    # oc get nodes
    NAME                STATUS    AGE       VERSION
    os-master1   Ready     15m       v1.6.1+5115d708d7
    os-node1     Ready     15m       v1.6.1+5115d708d7

## コンテナが名前解決に失敗する

コンテナは、Dockerホスト上で動作しているdnsmasqを読みに行く。dnsmasqの設定が正しく行われていないと
コンテナ上で名前解決に失敗する。masterノード上で下記のようなファイルを作成してやり

### /etc/dnsmasq.d/external-dns.conf

    server=127.0.0.1
    server=8.8.8.8

dnsmasqサービスを再起動する。

    # systemctl restart dnsmasq.service


### サービスカタログのインストールに失敗する


下記のようなメッセージが出力され、サービスカタログのインストールに失敗する場合、

    TASK [openshift_service_catalog : wait for api server to be ready] *************************************************************************************
    Saturday 23 June 2018  08:18:45 +0000 (0:00:01.207)       0:01:16.159 *********
    FAILED - RETRYING: wait for api server to be ready (60 retries left).
    FAILED - RETRYING: wait for api server to be ready (59 retries left).
    ...
    FAILED - RETRYING: wait for api server to be ready (1 retries left).
    fatal: [ose3-master1.test.example.com]: FAILED! => {"attempts": 60, "changed": false, "content": "", "msg": "Status code was -1 and not [200]: Request f     ailed: <urlopen error [Errno -2] Name or service not known>", "redirected": false, "status": -1, "url": "https://apiserver.kube-service-catalog.svc/healthz"}

マスターノード(ose3-master1)の/etc/resolve.confを下記ののように設定する。

    search cluster.local
    nameserver 127.0.0.1

また、/etc/origin/node/resolv.confでDNSサーバのアドレスが設定されていることを確認する。

   server=10.0.0.4

マスターノード上で下記のcurlコマンドを叩いてokが返ればOK。

    $ curl -k -L https://apiserver.kube-service-catalog.svc.cluster.local/healthz
    ok

サービスカタログからインストールを続行するには、下記のコマンドを実行する

    $ ansible-playbook -i ../openshift-allinone-ansible/inventory/hosts.inventory playbooks/openshift-service-catalog/config.yml