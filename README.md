# reloader-operator

Example using stakater/Reloader operator. 

This is useful for situations where a pod should automatically restart in the event a secret or configmap changes (such as a trusted CA certificate) since an application may only read in the mounted file at startup.

This also shows how to add a trusted CA to an OCP cluster. See <https://docs.openshift.com/container-platform/4.7/networking/configuring-a-custom-pki.html>.

## Install Operator Globally

```sh
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update
helm upgrade -i stakater stakater/reloader -n reloader-operator --create-namespace \
  --set reloader.isOpenshift=true \
  --set reloader.deployment.securityContext.runAsUser=null
```

## Install trusted-cabundle configmap and app

> the Deployment has the `configmap.reloader.stakater.com/reload: "trusted-cabundle"` annotation.

```sh
helm upgrade -i app helm/app -n reloader-operator-test --create-namespace
```

## Test adding an additional trust bundle to the cluster

> <https://docs.openshift.com/container-platform/4.7/networking/configuring-a-custom-pki.html>

Deploy example CA bundle configmap

```sh
helm upgrade -i additional-ca-bundle-ocp helm/additional-ca-bundle-ocp -n openshift-config
```

Patch the Proxy to add the trusted CA

```sh
oc patch proxy cluster --type merge -p '{"spec":{"trustedCA":{"name":"user-ca-bundle"}}}'
```

```sh
oc describe configmap trusted-cabundle -n reloader-operator-test | egrep "Test self-signed CA"
```

You should see the comment listed. The trusted-cabundle configmap automatically updates.

## Test automated reloading when the configmap changes

Make a change to the configmap user-ca-bundle in openshift-config. This should cause a new deployment of nginx-echo-headers in reloader-operator-test since the trusted-cabundle configmap also automatically updates.

```sh
oc status -n reloader-operator-test
```