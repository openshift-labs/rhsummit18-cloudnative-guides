# Red Hat Summit 2018: <br/>Getting Started with Cloud-Native Applications Lab Guide [![Build Status](https://travis-ci.org/openshift-labs/rhsummit18-cloudnative-guides.svg?branch=ocp-3.9)](https://travis-ci.org/openshift-labs/rhsummit18-cloudnative-guides)

Lab guides for the Cloud-Native Lab at Red Hat Summit 2018

Install Workshop Infrastructure
===

An [APB](https://hub.docker.com/r/openshiftapb/cloudnative-istio-workshop-apb) is provided for 
deploying the Cloud-Native with Istio Workshop infra (lab instructions, Nexus, Gogs, Eclipse Che, etc) in a project 
on an OpenShift cluster via the service catalog. In order to add this APB to the OpenShift service catalog, log in 
as cluster admin and perform the following in the `openshift-ansible-service-broker` project :

1. Edit the `broker-config` configmap and add this snippet right after `registry:`:

  ```
    - name: dh
      type: dockerhub
      org: openshiftapb
      tag: ocp-3.9
      white_list: [.*-apb$]
  ```

2. Redeploy the `asb` deployment

You can [read more in the docs](https://docs.openshift.com/container-platform/3.9/install_config/oab_broker_configuration.html#oab-config-registry-dockerhub) 
on how to configure the service catalog.

Note that if you are using the _OpenShift Workshop_ in RHPDS, this APB is already available in your service catalog.

As an alternative, you can also run the APB directly in a pod on OpenShift to install the workshop infra:

```
oc login
oc new-project lab-infra

oc run apb --restart=Never --image="openshiftapb/cloudnative-istio-workshop-apb:ocp-3.9" \
    -- provision -vvv \
    -e namespace=$(oc project -q) \
    -e openshift_token=$(oc whoami -t) \
    -e openshift_master_url=$(oc whoami --show-server) \
    -e user_count=10
```

Or if you have Ansible installed locally, you can also run the Ansilbe playbooks directly on your machine:

```
oc login
oc new-project lab-infra

ansible-playbook -vvv playbooks/provision.yml \
       -e namespace=$(oc project -q) \
       -e openshift_token=$(oc whoami -t) \
       -e openshift_master_url=$(oc whoami --show-server) \
       -e user_count=10 
``` 

Lab Instructions on OpenShift
===

Note that if you have used the above workshop installer, the lab instructions are already deployed.

```
$ oc new-app osevg/workshopper:latest --name=guides \
    -e WORKSHOPS_URLS="https://raw.githubusercontent.com/openshift-labs/cloud-native-guides/ocp-3.9/_cloud-native-roadshow.yml"
$ oc expose svc/guides
```

Local Lab Instructions
===
```
$ docker run -p 8080:8080 \
              -v $(pwd):/app-data \
              -e CONTENT_URL_PREFIX="file:///app-data" \
              -e WORKSHOPS_URLS="file:///app-data/_rhsummit18.yml" \
              osevg/workshopper:latest
```