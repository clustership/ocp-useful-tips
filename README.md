# Useful oc/OCP commands I usually used (and I have to look for quite often)

## Get informations using template or jsonpath

Get clusterID

```bash
oc get clusterversion version -o go-template='{{.spec.clusterID}}{{"\n"}}'
```


Get self-signed CA Cert

