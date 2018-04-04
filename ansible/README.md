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

For MAC OS X Users
-------------------

Know Issues and work arounds:

* Running this with the default python that is part of mac os x will fail, because it cannot verify the certificate to github.com. See https://blog.bearandgiraffe.com/ansible-github-and-a-failed-to-validate-the-ssl-certificate-story-a8a10ea17512 for more details on how to upgrade the ansible to use a later version of python.
* The ansible galaxy roles requires `jmespath` python package. To install it run: `sudo pip3 install jmespath`

