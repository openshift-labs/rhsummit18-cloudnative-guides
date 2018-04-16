#### Create Production Environment

TBD

~~~shell
oc create secret generic git-credentials --from-literal=username={{ GIT_USERNAME }} --from-literal=password={{ GIT_PASSWORD }}
oc label secret git-credentials credential.sync.jenkins.openshift.io=true
~~~