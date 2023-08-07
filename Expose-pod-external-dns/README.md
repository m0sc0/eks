# EXTERNAL DNS

En el ejemplo anterior creamos un ingress automaticamente pero la ruta con nuestro hostname la creamos a travez de un registro CNAME y manualmente. Ahora quiero crear esta ruta automaticamente cuando cree el ingress.

1) Creamos una zona host si es que no existe:
``` bash
aws route53 create-hosted-zone --name "external-dns-test.my-org.com." --caller-reference "external-dns-test-$(date +%s)"
```

2) Creamos este archivo de policy:
``` bash
cat <<EOF > hosted-zone-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

aws iam create-policy --policy-name HostedZoneIAMPolicy --policy-document file://hosted-zone-policy.json

{
    "Policy": {
        "PolicyName": "HostedZoneIAMPolicy",
        "PolicyId": "ANPAYU65SLJQPU3K3T3X7",
        "Arn": "arn:aws:iam::594780772960:policy/HostedZoneIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-08-07T11:15:33+00:00",
        "UpdateDate": "2023-08-07T11:15:33+00:00"
    }
}

```


3) Creamos la sa
``` bash
eksctl create iamserviceaccount  --name external-dns  --namespace kube-system  --cluster Giovanni --attach-policy-arn arn:aws:iam::594780772960:policy/HostedZoneIAMPolicy --approve
```

4) Creamos los recursos para el EXTERNAL DNS

``` yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:latest
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=sandbox302.opentlc.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        - --provider=aws
        - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
        - --registry=txt
        - --txt-owner-id=my-hostedzone-identifier
      securityContext:
        fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes and AWS token filesLoadBalancer
```