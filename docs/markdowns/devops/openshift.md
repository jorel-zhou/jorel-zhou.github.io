---
icon: simple/redhatopenshift
---

#### OpenShift CLI

[OpenShift CLI developer command reference](https://docs.openshift.com/container-platform/4.14/cli_reference/openshift_cli/developer-cli-commands.html)
[OpenShift CLI administrator command reference](https://docs.openshift.com/container-platform/4.14/cli_reference/openshift_cli/administrator-cli-commands.html)


```bash
oc login -u joel https://api.openshift.com:6443 --insecure-skip-tls-verify

oc config view
oc config set-context `oc config current-context` --namespace=<project_name>

oc whoami -c

oc status

oc new-project user-getting-started --display-name="Getting Started with OpenShift"

oc adm policy add-role-to-user view -z default -n user-getting-started

oc scale --current-replicas=1 --replicas=2 deployment/parksmap

oc projects |grep -v "openshift\|kube"

oc project myproject

oc get pods -o wide |grep Error | awk 'print $1, $2' | xargs -L1 oc delete pod

#将 Pod 从节点中逐出
oc drain [pod-name] 

# 封锁节点而不耗尽
oc cordon [node-name] --ignore-daemonsets 

#允许 Pod 返回节点
oc cordon [node-name] 

oc delete pods [pod-name] --grace-period=0 --force

oc plugin list
oc ns

oc logout

# Update pod 'foo' with the annotation 'description' and the value 'my frontend'
# If the same annotation is set multiple times, only the last value will be applied
oc annotate pods foo description='my frontend'

# Update a pod identified by type and name in "pod.json"
oc annotate -f pod.json description='my frontend'

# Update pod 'foo' with the annotation 'description' and the value 'my frontend running nginx', overwriting any existing value
oc annotate --overwrite pods foo description='my frontend running nginx'

# Update all pods in the namespace
oc annotate pods --all description='my frontend running nginx'

# Update pod 'foo' only if the resource is unchanged from version 1
oc annotate pods foo description='my frontend running nginx' --resource-version=1

# Update pod 'foo' by removing an annotation named 'description' if it exists
# Does not require the --overwrite flag
oc annotate pods foo description-

```