# 下準備 

下記のAnsibleを実行し、openshift-ansibleを実行する下準備を行う。

    # git clone https://github.com/okamototk/openshift-allinone-ansible
    # cd openshift-allinone-ansible

hostsを編集し、OpenShiftのサーバを指定する

    [openshift]
    os-master1
    os-node1

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

低スペックのマシンでテストする場合は、スペックチェックで引っかかるので
下記のファイルを編集し、チェックを無効にする。

### playbooks/common/openshift-cluster/config.yml

```
   - action: openshift_health_check
     args:
       checks:
#      - disk_availability
#      - memory_availability
       - package_availability
       - package_version
```

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
openshift_release=v3.6

[masters]
ose3-master1

[etcd]
os-master1

[nodes]
os-master1 openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
os-node1   openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
```

下記のコマンドでansibleを実行

    # ansible-playbook -i hosts.inventry playbooks/byo/config.yml

# トラブルシューティング


## masterノードが正しく動作しない

途中、routerやregistryの作成がうまくいかない場合は、スケジューラが正しく
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
## Docker Resgitryへのpushに失敗する

Docker Resgistryへのpushに下記のメッセージと共に失敗する

    error: build error: Failed to push image: Get https://docker-registry.default.svc:5000/v1/_ping: dial tcp: lookup docker-registry.default.svc on 168.63.129.16:53: no such host

ResgistryのURLがOpenShiftに登録されたレジストリとあっていない場合発声する。下記の記述を追加する。

### /etc/sysconfig/origin-master


    OPENSHIFT_DEFAULT_REGISTRY=docker-registry.default.svc.cluster.local:5000

## コンテナが名前解決に失敗する

コンテナは、Dockerホスト上で動作しているdnsmasqを読みに行く。dnsmasqの設定が正しく行われていないと
コンテナ上で名前解決に失敗する。masterノード上で下記のようなファイルを作成してやり

### /etc/dnsmasq.d/external-dns.conf

    server=127.0.0.1
    server=8.8.8.8

dnsmasqサービスを再起動する。

    # systemctl restart dnsmasq.service