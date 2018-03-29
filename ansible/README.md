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

How To Use
------------

```
ansible-galaxy install -r requirements.yml
ansible-playbook init.yml 
```
