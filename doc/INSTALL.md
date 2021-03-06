# Full CI/CD Setup

This section describe how to setup a complete CI/CD environment using Jenkins.
You can either setup the environment :
 - in a fully automated way, using OpenShift templates
 - manually

## Automated setup through Templates

Create the four environments :
```
oc new-project demo-dev
oc new-project demo-build
oc new-project demo-test
oc new-project demo-prod
```

Deploy a Jenkins in the BUILD project :
```
oc new-app -n demo-build --template=jenkins-persistent --name=jenkins -p MEMORY_LIMIT=1Gi
```

__Note :__ The `jenkins-persistent` Template requires you to provision a PersistentVolume.
If there is no available PersistentVolume, the deployment will fail. In this case, have a look to
my other project : the [OpenShift-Hostpath-Provisioner](https://github.com/nmasse-itix/OpenShift-HostPath-Provisioner).

Create all other objects using the template :
```
oc process -f setup/all-in-one-template.yaml TEST_ROUTE_HOSTNAME=demo.test.app.openshift.test PROD_ROUTE_HOSTNAME=demo.prod.app.openshift.test > objects.json
oc create -f objects.json
```

__Notes :__
 - Keep the `objects.json` in a safe place since you will need it to cleanup the
   platform (`oc delete -f objects.json`).
 - Replace the `demo.test.app.openshift.test` and `demo.prod.app.openshift.test`
   by meaningful values for your environment. It will be your routes in
   TEST and PROD environments.

All parameters are documented here :

| Parameter Name | Required ? | Default Value | Description |
| --- | --- | --- | --- |
| TEST_ROUTE_HOSTNAME | Yes | - | The route to create in the TEST environment and which we will use to run the integration tests |
| PROD_ROUTE_HOSTNAME | Yes | - | The route to create in the PROD environment |
| NPM_CREDENTIALS_ID | No | npm-secret | The Jenkins Credentials ID that holds the token to login on NPM Registry |
| GIT_REPO | No | https://github.com/nmasse-itix/OpenShift-Demo-NodeJS.git | The GIT repository to use. This will be useful if you clone this repo. |
| NODEJS_IMAGE_STREAM_TAG | No | nodejs:latest | Name of the ImageStreamTag to be used for the NodeJS image. Change this if you plan to use your own NodeJS S2I image. |
| NODEJS_IMAGE_STREAM_NAMESPACE | No | openshift | The OpenShift Namespace where the NodeJS ImageStream resides. |
| DEV_PROJECT | No | demo-dev | The name of the OpenShift Project to that holds the dev environment |
| BUILD_PROJECT | No | demo-build | The name of the OpenShift Project to that holds the build environment |
| TEST_PROJECT | No | demo-test | The name of the OpenShift Project to that holds the test environment |
| PROD_PROJECT | No | demo-prod | The name of the OpenShift Project to that holds the prod environment |

**Then, configure Jenkins [as described here](CONFIGURE_JENKINS.md).**

## Manual Setup

Create the four environments :
```
oc new-project demo-dev
oc new-project demo-build
oc new-project demo-test
oc new-project demo-prod
```

Create the build config in the BUILD environment :
```
oc new-build -n demo-build nodejs~https://github.com/nmasse-itix/OpenShift-Demo-NodeJS.git --strategy=source --name=openshift-demo-nodejs
```

You can test the build config with :
```
oc start-build -n demo-build openshift-demo-nodejs --env=NPM_PACKAGE_TO_INSTALL=openshift-demo-nodejs-nmasse@0.1.0
```

Deploy a Jenkins in the build project :
```
oc new-app -n demo-build --template=jenkins-persistent --name=jenkins -p MEMORY_LIMIT=1Gi
```

__Note :__ The `jenkins-persistent` Template requires you to provision a PersistentVolume.
If there is no available PersistentVolume, the deployment will fail. In this case, have a look to
my other project : the [OpenShift-Hostpath-Provisioner](https://github.com/nmasse-itix/OpenShift-HostPath-Provisioner).

Give admin role to the jenkins service account on subsequent environments :
```
oc adm policy add-role-to-user admin system:serviceaccount:demo-build:jenkins -n demo-test
oc adm policy add-role-to-user admin system:serviceaccount:demo-build:jenkins -n demo-prod
```

Tag the test image :
```
oc tag demo-build/openshift-demo-nodejs:latest openshift-demo-nodejs:ready-for-testing -n demo-test
```

Create the test application :
```
oc new-app demo-build/openshift-demo-nodejs:ready-for-testing --name openshift-demo-nodejs -n demo-test
oc expose service openshift-demo-nodejs --name=openshift-demo-nodejs --hostname=demo.test.app.openshift.test -n demo-test
```

Tag the prod image :
```
oc tag demo-test/openshift-demo-nodejs:ready-for-testing openshift-demo-nodejs:ready-for-prod -n demo-prod
```

Create the prod application (blue) :
```
oc new-app demo-build/openshift-demo-nodejs:ready-for-prod --name openshift-demo-nodejs-blue -n demo-prod -l color=blue
```

Create the prod application (green) :
```
oc new-app demo-build/openshift-demo-nodejs:ready-for-prod --name openshift-demo-nodejs-green -n demo-prod -l color=green
```

Create a service on selector "color" :
```
oc create -n demo-prod -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: openshift-demo-nodejs
  name: openshift-demo-nodejs
spec:
  ports:
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    color: blue
  sessionAffinity: None
  type: ClusterIP
EOF
```

And expose this service as a route :
```
oc expose service openshift-demo-nodejs --name=openshift-demo-nodejs --hostname=demo.prod.app.openshift.test -n demo-prod
```

You can verify the target of the route with :
```
oc get service openshift-demo-nodejs -n demo-prod -o template --template='{{ .spec.selector.color }}'
```

You can change the target of the route with :
```
oc patch -n demo-prod service/openshift-demo-nodejs --patch '{"spec":{"selector":{"color":"green"}}}'
```

Disable the triggers :
```
oc set triggers dc/openshift-demo-nodejs --from-image=demo-build/openshift-demo-nodejs:ready-for-testing --manual=true -c openshift-demo-nodejs -n demo-test
oc set triggers dc/openshift-demo-nodejs-blue --from-image=demo-build/openshift-demo-nodejs:ready-for-prod --manual=true -c openshift-demo-nodejs-blue -n demo-prod
oc set triggers dc/openshift-demo-nodejs-green --from-image=demo-build/openshift-demo-nodejs:ready-for-prod --manual=true -c openshift-demo-nodejs-green -n demo-prod
```

**Then, configure Jenkins [as described here](CONFIGURE_JENKINS.md).**
