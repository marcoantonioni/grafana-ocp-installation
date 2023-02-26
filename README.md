# Grafana OpenShift Installation

## Installation steps

### create destination namespace
```
TNS=grafana
oc new-project ${TNS}
oc project ${TNS}
```

### install OperatorGroup in namespace ONLY if no other already present
```
oc get operatorgroups.operators.coreos.com -n ${TNS}

OG_NAME=grafana-operator-group
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: ${OG_NAME}-
  name: ${OG_NAME}
  namespace: ${TNS}
spec:
  targetNamespaces:
  - ${TNS}
EOF
```

### install operator subscription
```
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/grafana-operator.grafana: ''
  name: grafana-operator
  namespace: ${TNS}
spec:
  channel: v4
  installPlanApproval: Automatic
  name: grafana-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
  startingCSV: grafana-operator.v4.9.0
EOF
```

### wait for resource created and status Succeeded
```
oc get csv -n ${TNS} | grep grafana-operator

oc get installplans.operators.coreos.com -n ${TNS}
```

### add grafana user to Thanos oauth htpasswd domain
```
THANOS_USER=thanos
THANOS_PASSWORD=passw0rd

# verify pod presence
oc get pod -n openshift-monitoring -lapp.kubernetes.io/instance=thanos-querier

# if Thanos is present, continue with next steps, otherwise halt the grafana installation
oc get secret -n openshift-monitoring thanos-querier-oauth-htpasswd -o jsonpath='{.data.auth}' | base64 -d > /tmp/thanos-htpasswd

echo "" >> /tmp/thanos-htpasswd
cat /tmp/thanos-htpasswd

# add user/password
htpasswd -s -b /tmp/thanos-htpasswd ${THANOS_USER} ${THANOS_PASSWORD}
cat /tmp/thanos-htpasswd

# update htpasswd
oc patch secret -n openshift-monitoring thanos-querier-oauth-htpasswd -p "{\"data\":{\"auth\":\"$(base64 -w0 /tmp/thanos-htpasswd)\"}}"

# verify patched secret
oc get secret -n openshift-monitoring thanos-querier-oauth-htpasswd -o jsonpath='{.data.auth}' | base64 -d

# restart thanos pods
oc get pod -n openshift-monitoring | grep thanos-querier | awk '{print $1}' | xargs oc delete pod -n openshift-monitoring

# verify with basic auth from thanos pod (if the authentication is failed you see lines '<title>Log In</title>' and '<button type="submit" class="btn btn-lg btn-default">Log In</button>')
curl -sk -u "thanos:passw0rd" https://thanos-querier.openshift-monitoring.svc.cluster.local:9091/api/datasources/proxy/1/api/v1/query_range | grep "Log In"

```

### create Prometheus datasource with basic authentication to Thanos
```
cat <<EOF | oc apply -f -
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: prometheus-grafanadatasource-basic
  namespace: ${TNS}
spec:
  datasources:
    - access: proxy      
      editable: true
      isDefault: true
      type: prometheus
      name: Prometheus-basic
      url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
      jsonData:
        timeInterval: 5s
        tlsSkipVerify: true
      secureJsonData:
        basicAuthPassword: ${THANOS_PASSWORD}
      basicAuth: true
      basicAuthUser: ${THANOS_USER}
  name: prometheus-grafanadatasource-basic.yaml
EOF
```

### install Grafana instance
```
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=passw0rd

cat <<EOF | oc apply -f -
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana
  namespace: ${TNS}
spec:
  config:
    auth.anonymous:
      enabled: true
    plugins:
      plugin_admin_enabled: true
    security:
      admin_user: ${GRAFANA_ADMIN_USER}
      admin_password: ${GRAFANA_ADMIN_PASSWORD}
      allow_embedding: true
    users:
      viewers_can_edit: true
    log:
      level: warn
      mode: console      
EOF
```

### create Route to access Grafana site, then login using GRAFANA_ADMIN_... credentials
```
oc expose service grafana-service --name=grafana
oc get routes -n ${TNS}
```


# References

## RedHat installation guide
https://www.redhat.com/en/blog/custom-grafana-dashboards-red-hat-openshift-container-platform-4


## Grafana dashboards
https://grafana.com/grafana/dashboards
