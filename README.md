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


OpenShiftの構成を定義したhosts.inventryを作成する

### hosts.inventry

```
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
ansible_ssh_user=root

product_type=openshift
deployment_type=origin
openshift_release=v3.7
openshift_master_api_port=443
openshift_master_console_port=443

openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/openshift/openshift-passwd'}]

[masters]
ose3-master1

[etcd]
os-master1

[nodes]
os-master1 openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
os-node1   openshift_node_labels="{'region': 'primary', 'zone': 'east'}"

```

下記のコマンドでansibleを実行

    # ansible-playbook -i inventory/hosts.myinventry playbooks/deploy_cluster.yml  -e openshift_disable_check=disk_availability,memory_availability

小さいサイズのVMなどで試す場合は、diskとmeomryチェックを無視する設定を追加する。

# 確認

OpenShiftにアクセスするWebブラウザのhostsファイルにose3-lbホストにアクセスできるIPアドレスをopenshift-ansible.public.example.comに設定する。Windowsの場合、下記のファイルに記述する。

C:\Windows\System32\drivers\etc\hosts

    ...
    23.96.13.5 openshift-ansible.public.example.com


https://openshift-ansible.public.example.com:8443にブラウザでアクセスする。ユーザadmin、パスワードsmartvmでログインし、アクセスできることを確認する。

# 使い方

masterノードにSSHでログインし、下記のコマンドでadminユーザ(パスワードadmin)を作成する。

    # htpasswd -b /etc/openshift/openshift-passwd  admin admin

adminユーザでログインする。

    # oc login
    Authentication required for https://os-master1.local:8443 (openshift)
    Username: admin
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


masterノード上でpingを実行し、名前解決ができていることを確認する。

    # ping docker-registry.default.svc
    PING docker-registry.default.svc.cluster.local (172.30.93.136) 56(84) bytes of data.

名前解決ができていなければ、

/etc/dnsmasq.d/external-dns.confに127.0.0.1が設定されていることと、

    server=127.0.0.1
    server=8.8.8.8

/etc/resolve.confでcluster.localをDNSの検索対象に入っていることを確認する。

    search cluster.local
    nameserver 192.168.1.151


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