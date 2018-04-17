#### Create Production Environment

TBD

~~~shell
oc create secret generic git-credentials --from-literal=username={{ GIT_USERNAME }} --from-literal=password={{ GIT_PASSWORD }}
oc label secret git-credentials credential.sync.jenkins.openshift.io=true
~~~

~~~shell
# oc policy add-role-to-user edit system:serviceaccount:dev:jenkins -n prod{{ PROJECT_SUFFIX }}
oc policy add-role-to-user system:deployer system:serviceaccount:dev:jenkins -n prod{{ PROJECT_SUFFIX }}
~~~
