# Useful oc/OCP commands I usually used (and I have to look for quite often)


## Find kernel version associated with OCP version

```bash
oc image info -a ~/.pull-secret.json --output json $(oc adm release info -a ~/.pull-secret.json --image-for=machine-os-content $(oc get clusterimagesets img4.12.7-x86-64-appsub -o jsonpath='{.spec.releaseImage}')) | jq .config.config.Labels | egrep coreos.rpm.kernel-rt-core
  "com.coreos.rpm.kernel-rt-core": "4.18.0-372.46.1.rt7.203.el8_6.x86_64",
  

## Get informations using template or jsonpath

Get clusterID

```bash
oc get clusterversion version -o go-template='{{.spec.clusterID}}{{"\n"}}'
```


Get self-signed CA Cert


# Create a pod with tmpfs emptyDir (mountOptions unfortunaltely does not work)

```bash
cat <<EOF > tmpfs-validation.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: tmpfs-validation
  name: tmpfs-validation
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tmpfs-validation
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: tmpfs-validation
    spec:
      containers:
      - image: quay.io/xymox/ubi8-debug-toolkit:latest
        name: ubi8-debug-toolkit
        resources: {}
        command:
        - /bin/sh
        - -c
        - sleep infinity
        volumeMounts:
        - name: tmp-storage
          mountPath: /data/tmp
      volumes:
      - name: tmp-storage
        emptyDir:
          medium: Memory
          mountOptions:
          - size=100
          - noexec
          - nosuid
status: {}
EOF
oc apply -f tmp-validation.yml
```


# Get events in the right order

```bash
 oc -n openshift-marketplace get events --sort-by='.lastTimestamp'
 ```
 
 # Get self-signed OCP certificate
 
 WIP
 
 ```bash
 oc get secret signing-key -n openshift-service-ca -o jsonpath='{.data.tls\.crt}' | base64 -d > ocp-ca.pem
 
 echo quit | openssl s_client -showcerts -servername server -connect api.osp-ocp4-07.clustership.com:6443 > cacert.pem
 ```
 
It does not work with curl.

See: https://stackoverflow.com/questions/17597457/why-wont-curl-recognise-a-self-signed-ssl-certificate/21262787#21262787
