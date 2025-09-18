# Exposing a MongoDB CE in Deployed in OCP

This technote deals with exposing a MongoDB CE instance used as a dependency of MAS deployed in Red Hat OpenShift.

This was tested using MongoDB CE v7.0.22, used by MAS v9.0.14, ibm-operator-catalog version v9-250902-amd64, OCP v4.16.45. This Mongo instance is configured to be used as an external mongodb for another MAS instance v9.1.x, using the same ibm-operator-catalog version in OCP 4.16.x.

Ref:https://www.linkedin.com/pulse/expose-mongdb-community-edition-redhat-openshift-daniel-istrate/ 

1. Create the services and routes for the mongodb replicas. Apply the yamls separately.

#### Services

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mas-mongo-ce-0
  namespace: mongoce
spec:
  ports:
    - name: mongodb
      protocol: TCP
      port: 443
      targetPort: 27017
  selector:
    statefulset.kubernetes.io/pod-name: mas-mongo-ce-0
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mas-mongo-ce-1
  namespace: mongoce
spec:
  ports:
    - name: mongodb
      protocol: TCP
      port: 443
      targetPort: 27017
  selector:
    statefulset.kubernetes.io/pod-name: mas-mongo-ce-1
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mas-mongo-ce-2
  namespace: mongoce
spec:
  ports:
    - name: mongodb
      protocol: TCP
      port: 443
      targetPort: 27017
  selector:
    statefulset.kubernetes.io/pod-name: mas-mongo-ce-2
```

#### Routes 
*Note: update the domain accordingly

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mas-mongo-ce-0
  namespace: mongoce
spec:
  host: mas-mongo-ce-0.mongoce.apps.example.com
  to:
    kind: Service
    name: mas-mongo-ce-0
  tls:
    termination: passthrough
```
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mas-mongo-ce-1
  namespace: mongoce
spec:
  host: mas-mongo-ce-1.mongoce.apps.example.com
  to:
    kind: Service
    name: mas-mongo-ce-1
  tls:
    termination: passthrough
```
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: mas-mongo-ce-2
  namespace: mongoce
spec:
  host: mas-mongo-ce-2.mongoce.apps.example.com
  to:
    kind: Service
    name: mas-mongo-ce-2
  tls:
    termination: passthrough
```
2. Copy the hosts from these external routes.

3. Update the certificates and secret to add the external hostnames.

Go to Home > Search in OpenShift console page, select the mongoce project and search for resource type Certificate. The following certificates will be listed:

- mongo-ca-crt
- mongo-server

Modify mongo-ca-crt yaml spec.dnsNames list with the following (remember to modify the domain accordingly).

```yaml
spec:
  commonName: mongo-ca-crt
  dnsNames:
    - '*.mas-mongo-ce-svc.mongoce.svc.cluster.local'
    - 127.0.0.1
    - localhost
    - mas-mongo-ce-0.mongoce.svc.cluster.local
    - mas-mongo-ce-1.mongoce.svc.cluster.local
    - mas-mongo-ce-2.mongoce.svc.cluster.local
    - mas-mongo-ce-0.mongoce.apps.example.com
    - mas-mongo-ce-1.mongoce.apps.example.com
    - mas-mongo-ce-2.mongoce.apps.example.com
```
Do the same for mongo-server.

4. Get the ca certificate from the mongo-server-cert:

```
oc extract secret/mongo-server-cert -n mongoce --keys=ca.crt --to=- > ca.crt
```

5. Update the ConfigMap mas-mongo-ce-cert-map with the new ca.crt you got from the preceding step.

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: mas-mongo-ce-cert-map
  namespace: mongoce
  managedFields:
<snip>
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    ....
    -----END CERTIFICATE-----
```

6. Restart the mas-mongo-ce pods.

7. Update the mongoDBCommunity CR's spec.replicaSetHorizons:

*Update the domain accordingly.

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: ''
    mongodb.com/v1.lastAppliedMongoDBVersion: 7.0.22
    mongodb.com/v1.lastSuccessfulConfiguration: ''
  name: mas-mongo-ce
  namespace: mongoce
spec:
  security:
    authentication:
      ignoreUnknownUsers: true
      modes:
        - SCRAM-SHA-256
        - SCRAM-SHA-1
    tls:
      caConfigMapRef:
        name: mas-mongo-ce-cert-map
      certificateKeySecretRef:
        name: mongo-server-cert
      enabled: true
  replicaSetHorizons:
    - ocroute: 'mas-mongo-ce-0.mongoce.apps.ilangilang.cp.fyre.ibm.com:443'
      ocsvc: 'mas-mongo-ce-0.mongoce.svc.cluster.local:443'
    - ocroute: 'mas-mongo-ce-1.mongoce.apps.ilangilang.cp.fyre.ibm.com:443'
      ocsvc: 'mas-mongo-ce-1.mongoce.svc.cluster.local:443'
    - ocroute: 'mas-mongo-ce-2.mongoce.apps.ilangilang.cp.fyre.ibm.com:443'
      ocsvc: 'mas-mongo-ce-2.mongoce.svc.cluster.local:443'
  users:
<snip>
  featureCompatibilityVersion: '7.0'
```

8. Wait for the mongoDBcommunity CR to become 'READY'.

9. Test if you can connect to the mongodb replicas externally. You can use any tool to do this. Please follow the instructions in https://www.linkedin.com/pulse/expose-mongdb-community-edition-redhat-openshift-daniel-istrate/

Data needed:
- hosts and port
- mongodb credentials
- mongo cert

