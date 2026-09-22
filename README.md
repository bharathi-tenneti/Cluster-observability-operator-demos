# Cluster-observability-operator-demos
Set up quickly COO alerting and logging for Openshift 4.22

## coo-logging setup order

Apply in this order — later steps depend on CRDs/resources created by earlier ones.

```bash
# 1. Operators
oc apply -f coo-logging/01-cluster-observability-operator.yaml
oc apply -f coo-logging/02-loki-operator.yaml
oc apply -f coo-logging/03-cluster-logging-operator.yaml

# 2. Object storage bucket claim (wait for PHASE=Bound before continuing)
oc apply -f coo-logging/04-loki-object-bucket-claim.yaml
oc get obc loki-bucket-odf -n openshift-logging -w

# 3. Translate the OBC's generated credentials into the logging-loki-s3 secret
#    that LokiStack expects (different secret name + key names than what ODF/NooBaa
#    generates automatically). Must run before step 4.
#    Read the values first, fill them into coo-logging/04b-loki-s3-secret.yaml,
#    then apply:
oc get cm loki-bucket-odf -n openshift-logging -o yaml
oc get secret loki-bucket-odf -n openshift-logging -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d
oc get secret loki-bucket-odf -n openshift-logging -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d
# edit coo-logging/04b-loki-s3-secret.yaml with the values above, then:
oc apply -f coo-logging/04b-loki-s3-secret.yaml

# 4. LokiStack and its storage-serving-CA trust bundle
oc apply -f coo-logging/05-loki-storage-ca-configmap.yaml
oc apply -f coo-logging/06-lokistack.yaml
oc get lokistack logging-loki -n openshift-logging -w

# 5. Collector RBAC and log forwarding
oc apply -f coo-logging/07-collector-rbac.yaml
oc apply -f coo-logging/08-clusterlogforwarder.yaml

# 6. Console UI plugins
oc apply -f coo-logging/09-uiplugin-logging.yaml
oc apply -f coo-logging/10-uiplugin-monitoring.yaml
```

Verify: `oc get pods -n openshift-logging` should show the collector daemonset
plus LokiStack's gateway/distributor/ingester/querier/query-frontend/compactor/
index-gateway pods all `Running`, and the LokiStack `Degraded` condition cleared.

