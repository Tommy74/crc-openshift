# crc-openshift

## CRC

So you've installed [Code Ready Containers](https://developers.redhat.com/products/codeready-containers/overview) (= OpenShift for developers) and you've created a project named "myproject" to play around ...

```
crc setup
crc start 
oc login -u kubeadmin -p <SOME_PASSWORD> https://api.crc.testing:6443
oc new-project myproject
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
    sudo podman login -u="my-quay-org+myquayrobot" -p="AFL44C0L1AOXF5C70FK363F17NH4MRPTW7OYRVIA15LDXWWY66SYPIBO3QCL8ORO" quay.io
    ```
* use the "Kubernetes Secret" of "myquayrobot" to create a secret in CRC
    ```
    oc create -f ~/Downloads/my-quay-org-myquayrobot-secret.yml --namespace=myproject
    ```

# Create Image

Now use the **Dockerfile** to create an image and upload it to [quay.io](https://quay.io/):

```
sudo podman build . -t quay.io/my-quay-org/my-quay-repo
sudo podman push quay.io/my-quay-org/my-quay-repo
```

# Run the image in CRC

```
oc secrets link default my-quay-org-myquayrobot-pull-secret --for pull
oc new-app --docker-image=quay.io/my-quay-org/my-quay-repo
oc get pods
oc logs my-quay-repo-1-hkdlb
```

Note the name "my-quay-repo-1-hkdlb" was derived from the "oc get pods" output

