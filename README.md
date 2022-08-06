# Bootstrap Secrets Management

## Thycotic Secret Server

The following steps are intended for the use of Thycotic Secret Server with a mutating admission webhook on the Openshift cluster. This deployment model will allow for the use of dummy `Secrets` with GitOps that are replaced with real `Secret` values in the admission phase of a resource's lifecycle.

Set shell variables to be used for processing the templates:

```
USERNAME=<Thycotic Username>
PASSWORD=<Thycotic Password>
SERVER_URL=<Thycotic Server URL>
REPLICAS=1
SERVICE=tss-injector
NAMESPACE=thycotic
SIGNER_NODE_NAME=<node name from cluster>
```

Generate server key and CSR:

```
openssl genrsa -out /tmp/server-key.pem 2048

cat csr.conf.template | SERVICE=$SERVICE NAMESPACE=$NAMESPACE envsubst > /tmp/csr.conf

openssl req -new -key /tmp/server-key.pem -subj "/CN=system:node:${SIGNER_NODE_NAME};/O=system:nodes" -out /tmp/server.csr -config /tmp/csr.conf
```

_Note: Subject CN needs to specify a cluster node for `certificates.k8s.io/v1` of CertificateSigningRequest. https://github.com/kubernetes/kubernetes/issues/99504_

Create the project for Thycotic:

```
oc new-project $NAMESPACE
```

Process the template to create resources:

```
oc process -f thycotic.yaml \
    USERNAME=$USERNAME \
    PASSWORD=$PASSWORD \
    SERVER_URL=$SERVER_URL \
    INJECTOR_REPLICAS=$REPLICAS \
    SERVICE=$SERVICE \
    NAMESPACE=$NAMESPACE \
    CSR_BASE64=$(cat /tmp/server.csr | base64 | tr -d '\n') \
    | oc apply -f -
```

Approve the CSR and create a tls secret:

```
oc adm certificate approve thycotic

oc wait --for=condition=approved csr/thycotic

oc get csr thycotic --template={{.status.certificate}} | base64 -d > /tmp/server-cert.crt

oc create secret tls tss-cert \
  -n $NAMESPACE \
  --cert=/tmp/server-cert.crt \
  --key=/tmp/server-key.pem

oc rollout restart deployment $SERVICE -n $NAMESPACE
```

Create the webhook:

```
oc process -f thycotic-webhook.yaml \
    SERVICE=$SERVICE \
    NAMESPACE=$NAMESPACE \
    CA_BUNDLE_BASE64=$(oc get configmap -n kube-system extension-apiserver-authentication -o=jsonpath='{.data.client-ca-file}' | base64 | tr -d '\n') \
    | oc apply -f -
```

## Using Cert Manager

### Install Cert Manager with OLM

```
oc apply -f cert-manager/install.yaml
```

### Create the ClusterIssuer and Certificate

This will create a self-signed `ClusterIssuer` and a `Certificate` in the namespace of the injector service:

```
oc apply -f cert-manager/certificate.yaml
```

### Update the Deployment to use the Certificate Secret

Edit the Deployment to use the `tss-cert` secret in the injector namespace.

_TODO: add patch command_

```
oc patch ...
```

### Apply MutatingWebhookConfiguration

This will apply the webhook using a cert-manager annotation to inject the CA:

```
oc apply -f cert-manager/thycotic-webhook.yaml
```

### Creating a secret

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: test-secret
  namespace: test
  annotations:
    tss.thycotic.com/set-secret: '3'
  labels:
    injector: thycotic
type: Opaque
data:
  placeholder: cGxhY2Vob2xkZXI=
```

#### Available Secret Annotations

- `tss.thycotic.com/role` - identifies the role that should be used, as per the roles.json entry.
- `tss.thycotic.com/set-secret` - adds missing fields without overwriting or removing existing fields.
- `tss.thycotic.com/add-to-secret` - adds and overwrites existing fields but does not remove fields.
- `tss.thycotic.com/update-secret` - overwrites fields and removes fields that do not exist in the TSS Secret.

_Note: Only one of these should be specified on any given k8s Secret, however, if more than one are defined then the order of precedence is setAnnotation then addAnnotation then updateAnnotation._

### References

The following guide was used from Thycotic with modifications:
- https://docs.thycotic.com/ssi/1.0.0/ibm/redhat/openshift/secretserver.md

An excellent resource for describing configurations using Mutating Admission Webhooks:
- https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74
- https://www.scottguymer.co.uk/post/kubernetes-mutating-webhook-configuration/
- https://medium.com/ovni/writing-a-very-basic-kubernetes-mutating-admission-webhook-398dbbcb63ec

Resources used to troubleshoot certificate signing on cluster:
- https://stackoverflow.com/questions/65587904/condition-failed-attempting-to-approve-csrs-with-certificates-k8s-io-v1
- https://github.com/kubernetes/kubernetes/issues/99504