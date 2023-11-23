---
icon: simple/redhatopenshift
---

#### OpenShift CLI

```bash
oc login -u joel https://api.openshift.com:6443
oc projects |grep -v "openshift\|kube"
oc project myproject
```