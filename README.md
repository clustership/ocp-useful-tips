# Useful oc/OCP commands I usually used (and I have to look for quite often)

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


