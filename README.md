# crc-openshift

## CRC

So you've installed [Code Ready Containers](https://developers.redhat.com/products/codeready-containers/overview) (= OpenShift for developers) and you've created a project named "myproject" to play around ...

```
$ crc setup
$ crc start 
$ oc login -u kubeadmin -p <SOME_PASSWORD> https://api.crc.testing:6443
$ oc new-project myproject
```

## quay.io

Then you need some container registry to save the images you build ... then:

* create an account on [quay.io](/quay.io) *
* create an "Organization" named "my-quay-org"
* create a public repository named "my-quay-repo"
* create a "Robot Account" named "myquayrobot"
* grant "write" permissions to "myquayrobot" on "my-quay-repo"
* login to your repo using podman using the "Docker Login" credentials of "myquayrobot" (on my Fedora):
    ```
    $ sudo podman login -u="my-quay-org+myquayrobot" -p="AFL44C0L1AOXF5C70FK363F17NH4MRPTW7OYRVIA15LDXWWY66SYPIBO3QCL8ORO" quay.io
    ```
* use the "Kubernetes Secret" of "myquayrobot" to create a secret in CRC
    ```
    $ oc create -f ~/Downloads/my-quay-org-myquayrobot-secret.yml --namespace=myproject
    ```

## Create Image

Now use the **Dockerfile** to create an image and upload it to [quay.io](https://quay.io/):

```
$ sudo podman build . -t quay.io/my-quay-org/my-quay-repo
$ sudo podman push quay.io/my-quay-org/my-quay-repo
```

## Run the image in CRC

```
$ oc secrets link default my-quay-org-myquayrobot-pull-secret --for pull
$ oc new-app --docker-image=quay.io/my-quay-org/my-quay-repo
$ oc get pods
$ oc logs my-quay-repo-1-hkdlb
```

Note the name "my-quay-repo-1-hkdlb" was derived from the "oc get pods" output

## Config Maps and Secrets

We add some secrets as files into the DeploymentConfig:

```
$ echo "value1" > /tmp/key1.txt
$ echo "value2" > /tmp/key2.txt
$ echo "value3" > /tmp/key3.txt
$ oc create secret generic sec-files --from-file /tmp/key1.txt --from-file /tmp/key2.txt --from-file /tmp/key3.txt
$ oc set volume dc/my-quay-repo --add -t secret -m /opt/app-root/sec-files --name myappsec-files --secret-name sec-files
$ oc rsh my-quay-repo-5-z9pl2
sh-4.4$ ls -l /opt/app-root/sec-files
total 0
lrwxrwxrwx. 1 root root 15 Dec  8 08:57 key1.txt -> ..data/key1.txt
lrwxrwxrwx. 1 root root 15 Dec  8 08:57 key2.txt -> ..data/key2.txt
lrwxrwxrwx. 1 root root 15 Dec  8 08:57 key3.txt -> ..data/key3.txt
sh-4.4$ cat /opt/app-root/sec-files/*
value1
value2
value3
```

We do the same with a config map, but this time we add it to DeploymentConfig as env variables:

```
$ oc create configmap test-config-map --from-literal CHIAVE1=VALORE1 --from-literal CHIAVE2=VALORE2
$ oc patch configmap/test-config-map --patch '{"data":{"CHIAVE2":"NUOVOVALORE2"}}'
$ oc get configmap/test-config-map -o json
$ oc set env dc/my-quay-repo --from configmap/test-config-map
$ oc rsh my-quay-repo-6-p6k7n
sh-4.4$ echo $CHIAVE2
NUOVOVALORE2
sh-4.4$ env
_=/usr/bin/env
HOSTNAME=my-quay-repo-6-p6k7n
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=172.25.0.1
container=oci
KUBERNETES_PORT=tcp://172.25.0.1:443
PWD=/
HOME=/
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP=tcp://172.25.0.1:443
CHIAVE1=VALORE1
CHIAVE2=NUOVOVALORE2
TERM=xterm-256color
NSS_SDB_USE_CACHE=no
SHLVL=2
KUBERNETES_SERVICE_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_SERVICE_HOST=172.25.0.1
```