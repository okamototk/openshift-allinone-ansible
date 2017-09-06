# All in One OpenShift Install Directory

## Before installation

   $ sudo yum install -y epel-release
   $ sudo yum install -y git ansible

## Install

   $ git clone https://github.com/okamototk/openshift-allinone-ansible
   $ cd openshift-allinone-ansible/
   $ sudo ansible site.yml

## After installation

Set envirnoment (keeping them on .bash_profile is better)

	# export KUBECONFIG=/var/lib/origin/openshift.local.config/master/admin.kubeconfig
	# export CURL_CA_BUNDLE=/var/lib/origin/openshift.local.config/master/ca.crt
	# export PATH=/opt/openshift:${PATH}

Copy config file

    # cp /var/lib/origin/openshift.local.config/master/admin.kubeconfig  ~/.kube/config

Currently, support only root user.


## Test


    $ oc login -u admin:admin
    $ oc new project test
    $ oc project test
    $ oc new-app codecentric/springboot-maven3-centos~https://github.com/codecentric/springboot-sample-app.git
    $ oc expose service springboot-sample-app --hostname=springboot-sample-app.your-openshift-installation.com




