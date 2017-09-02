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

    # export KUBECONFIG=/opt/openshift/openshift.local.config/master/admin.kubeconfig
    # export CURL_CA_BUNDLE=/opt/openshift/openshift.local.config/master/ca.crt
    # export PATH=/opt/openshift:${PATH}

Copy config file

    # cp /opt/openshift/openshift.local.config/master/admin.kubeconfig  ~/.kube/config

Currently, support only root user.

Run private docker registry

    $ oc login -u system:admin
    $ oc project default
    $ oadm registry --service-account=registry --images='registry.access.redhat.com/openshift3/ose-${component}:${version}' --selector='region=infra'


