# Use a service account for automation

2. Instead of using a user identity, can we create a service account for each cluster we want to mange and create bound service account tokens for this serviceaccount with long expiry time than the cluster default? If possible, what is the maximum expiry time for that token

For automation, it is better to use a Serviceaccount than a user. Bound Service Account Token Projection is made for in cluster use, i.e. Kubernetes will rotate tokens in a projected volume automatically.
I understand that you would like to access the Kubernetes API from an external host, volume projection does not work here obviously.

You can skip to 3. if you are not interested in the legacy service account tokens which still works but will be removed soon. I tested on 4.12 Release Candidate 1.

$ oc create sa test
serviceaccount/test created
$ oc get secret
NAME                       TYPE                                  DATA   AGE
builder-dockercfg-9px7c    kubernetes.io/dockercfg               1      29h
builder-token-hvwjx        kubernetes.io/service-account-token   4      29h
default-dockercfg-d7gxk    kubernetes.io/dockercfg               1      29h
default-token-cv62l        kubernetes.io/service-account-token   4      29h
deployer-dockercfg-mjfqg   kubernetes.io/dockercfg               1      29h
deployer-token-w8s7g       kubernetes.io/service-account-token   4      29h
test-dockercfg-nw247       kubernetes.io/dockercfg               1      8s
test-token-826gh           kubernetes.io/service-account-token   4      8s
$ TOKEN=$(oc get secret test-token-826gh -o jsonpath={.data.token} | base64 -d)

$ curl -k -H "Authorization:Bearer $TOKEN" https://api.ocp412rc1.lab:6443/apis/node.k8s.io
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "node.k8s.io",
  "versions": [
    {
      "groupVersion": "node.k8s.io/v1",
      "version": "v1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "node.k8s.io/v1",
    "version": "v1"
  }
}

$ echo $TOKEN | jwt decode -

Token header
------------
{
  "alg": "RS256",
  "kid": "CdICdjqh3vnmc_qBBVHnhMn6N8_q0B22_cE8CUwrxBo"
}

Token claims
------------
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "test",
  "kubernetes.io/serviceaccount/secret.name": "test-token-826gh",
  "kubernetes.io/serviceaccount/service-account.name": "test",
  "kubernetes.io/serviceaccount/service-account.uid": "ac352032-2d84-4050-b5fb-4f1a03a80085",
  "sub": "system:serviceaccount:test:test"
}

You can see that the JWT has no "exp": which is short for expiry. I used jwt from https://github.com/mike-engel/jwt-cli

You would need to also configure RBAC for your use case, here is the entry point to the Openshift RBAC documentation:

https://docs.openshift.com/container-platform/4.11/authentication/using-rbac.html

3. Do you have any other suggestion rather than two above, in order to create kubeconfig file (long live) for the identity bound to a less privilege cluster role?

Bound Service Account Tokens were introduced (Around k8s 1.20 IIRC) to get greater control and expiry of service account tokens and this is a good practice.

While the above commands work for now, it will change soon
https://docs.openshift.com/container-platform/4.11/authentication/using-service-accounts-in-applications.html

In the future tokens will not be generated automatically and the TokenRequest API has to be used if a token is required.

You can create tokens with at least 10 years duration, even though I highly recommend you to think about token rotation for security reasons. The below assumes that there is a service account already created.

$ TOKEN=$(oc create token test --duration=87600h)
$ curl -k -H "Authorization:Bearer $TOKEN" https://api.ocp412rc1.lab:6443/apis/node.k8s.io
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "node.k8s.io",
  "versions": [
    {
      "groupVersion": "node.k8s.io/v1",
      "version": "v1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "node.k8s.io/v1",
    "version": "v1"
  }
$ echo $TOKEN | jwt decode -

Token header
------------
{
  "alg": "RS256",
  "kid": "xcS3k4Vptqk-CNNgxv2p7y5it2R7ZU-4kFTwf0HpOQg"
}

Token claims
------------
{
  "aud": [
    "https://kubernetes.default.svc"
  ],
  "exp": 1984583795,
  "iat": 1669223795,
  "iss": "https://kubernetes.default.svc",
  "kubernetes.io": {
    "namespace": "test",
    "serviceaccount": {
      "name": "test",
      "uid": "ac352032-2d84-4050-b5fb-4f1a03a80085"
    }
  },
  "nbf": 1669223795,
  "sub": "system:serviceaccount:test:test"
}
$ date -d @1984583795
Sat Nov 20 06:16:35 PM CET 2032
There are many tutorials on the internet on how to create a kubeconfig from a token that you can use.

One way of doing it is to take a backup of your kubeconfig, create a new context and delete the users and contexts you don't need from the kubeconfig. Something like:

$ cp ~/.kube/config ~/
$ oc config set-credentials sa-user --token=$TOKEN
$ oc config get-users
...
$ oc config delete-user XXXX <- Repeat for all users except sa-user
$ oc config get-contexts
...
$ oc config delete-context YYYY <- Repeat for all contexts except the context you want
$ cat ~/.kube/config <- Review the new kubeconfig, make a copy of it somewhere
...
$ mv ~/config ~/.kube/config <- restore you original kubeconfig

//David
