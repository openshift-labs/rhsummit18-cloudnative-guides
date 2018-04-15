Ansible Playbook For Workshop Setup
=========

The provided playbook automates preparing an OpenShift cluster for the Getting Started 
with Cloud-Native Labs by deploying required services (Gogs, Nexus, Eclipse Che, etc) which 
are used during the labs.

Playbook Variables
------------

| Variable              | Default Value | Description   |
|-----------------------|---------------|---------------|
|`lab_infra_project`    | `lab-infra`   | Project name to deploy Git server and lab guides  |
|`user_gogs_admin`      | `gogs`        | Admin username to create in Gogs |
|`user_gogs_test`       | `test`        | Test username to create in Gogs |
|`user_gogs_password`   | `openshift`   | Gogs password to configure for admin and test users |

How To Run
------------

```
ansible-galaxy install -r requirements.yml
ansible-playbook init.yml 
```

Troubleshooting 
---------------
* Mac OS X users need to upgrade the default Python installed to avoid certificate issues with GitHub. See https://blog.bearandgiraffe.com/ansible-github-and-a-failed-to-validate-the-ssl-certificate-story-a8a10ea17512 for more details on how to upgrade the ansible to use a later version of python.
* The Ansible Galaxy roles requires `jmespath` python package on Mac OS X. To install it run: `sudo pip3 install jmespath`



Tips
----------------
* Pre-pull images on all nodes

  ```
  docker pull registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.2
  dockerpull registry.access.redhat.com/rhscl/nodejs-4-rhel7:latest
  docker pull registry.access.redhat.com/openshift3/jenkins-2-rhel7:latest
  docker pull registry.access.redhat.com/openshift3/jenkins-slave-maven-rhel7:latest
  docker pull registry.access.redhat.com/rhscl/postgresql-95-rhel7:latest
  docker pull registry.access.redhat.com/rhscl/postgresql-96-rhel7:latest
  docker pull sonatype/nexus3:3.7.1
  docker pull openshiftdemos/gogs:0.11.34
  docker pull eclipse/che:6.3.0
  docker pull osevg/workshopper:latest
  docker pull siamaksade/centos_jdk8
  docker pull siamaksade/rhsummit18-cloudnative-inventory
  docker pull siamaksade/rhsummit18-cloudnative-web
  ```

* Add an admin user to the cluster. Run the following as `system:admin`:

  ```
  oc adm policy add-cluster-role-to-user cluster-admin admin
  ```
