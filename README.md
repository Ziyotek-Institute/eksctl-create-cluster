# Setting up ExternalDNS for Services on AWS

This tutorial describes how to setup ExternalDNS for usage within a Kubernetes cluster on AWS. Make sure to use **>=0.11.0** version of ExternalDNS for this tutorial

## IAM Policy

The following IAM Policy document allows ExternalDNS to update Route53 Resource
Record Sets and Hosted Zones. You'll want to create this Policy in IAM first. In
our example, we'll call the policy `AllowExternalDNSUpdates` (but you can call
it whatever you prefer).

If you prefer, you may fine-tune the policy to permit updates only to explicit
Hosted Zone IDs.

```json
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
```

If you are using the AWS CLI, you can run the following to install the above policy (saved as `policy.json`).  This can be use in subsequent steps to allow ExternalDNS to access Route53 zones.


```bash
aws iam create-policy --policy-name "AllowExternalDNSUpdates" --policy-document file://policy.json

# example: arn:aws:iam::XXXXXXXXXXXX:policy/AllowExternalDNSUpdates
export POLICY_ARN=$(aws iam list-policies \
 --query 'Policies[?PolicyName==`AllowExternalDNSUpdates`].Arn' --output text)
```

## Provisioning a Kubernetes cluster

You can use [eksctl](https://eksctl.io) to easily provision an [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks) ([EKS](https://aws.amazon.com/eks)) cluster that is suitable for this tutorial.  See [Getting started with Amazon EKS – eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html).


```bash
export EKS_CLUSTER_NAME="my-externaldns-cluster"
export EKS_CLUSTER_REGION="us-east-2"
export KUBECONFIG="$HOME/.kube/${EKS_CLUSTER_NAME}-${EKS_CLUSTER_REGION}.yaml"

eksctl create cluster --name $EKS_CLUSTER_NAME --region $EKS_CLUSTER_REGION
```

Feel free to use other provisioning tools or an existing cluster.  If [Terraform](https://www.terraform.io/) is used, [vpc](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/) and [eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/) modules are recommended for standing up an EKS cluster.  Amazon has a workshop called [Amazon EKS Terraform Workshop](https://tf-eks-workshop.workshop.aws/) that may be useful for this process.

## Permissions to modify DNS zone

You will need to use the above policy (represented by the `POLICY_ARN` environment variable) to allow ExternalDNS to update records in Route53 DNS zones. Here are three common ways this can be accomplished:

* [Node IAM Role](#node-iam-role)
* [Static credentials](#static-credentials)
* [IAM Roles for Service Accounts](#iam-roles-for-service-accounts)

For this tutorial, ExternalDNS will use the environment variable `EXTERNALDNS_NS` to represent the namespace, defaulted to `default`.  Feel free to change this to something else, such `externaldns` or `kube-addons`.  Make sure to edit the `subjects[0].namespace` for the `ClusterRoleBinding` resource when deploying ExternalDNS with RBAC enabled.  See [Manifest (for clusters with RBAC enabled)](#manifest-for-clusteres-with-rbac-enabled)  for more information.

### Node IAM Role

In this method, you can attach a policy to the Node IAM Role.  This will allow nodes in the Kubernetes cluster to access Route53 zones, which allows ExternalDNS to update DNS records.  Given that this allows all containers to access Route53, not just ExternalDNS, running on the node with these privileges, this method is not recommended, and is only suitable for limited limited test environments.

If you are using eksctl to provision a new cluster, you add the policy at creation time with:

```bash
eksctl create cluster --external-dns-access \
  --name $EKS_CLUSTER_NAME --region $EKS_CLUSTER_REGION \
```


:warning: **WARNING**: This will assign allow read-write access to all nodes in the cluster, not just ExternalDNS.  For this reason, this method is only suitable for limited test environments.

If you already provisioned a cluster or use other provisioning tools like Terraform, you can use AWS CLI to attach the policy to the Node IAM Role.

#### Get the Node IAM role name

The role name of the role associated with the node(s) where ExternalDNS will run is needed.  An easy way to get the role name is to use the AWS web console (https://console.aws.amazon.com/eks/), and find any instance in the target node group and copy the role name associated with that instance.

##### Get role name with a single managed nodegroup

From the command line, if you have a single managed node group, the default with `eksctl create cluster`, you can find the role name with the following:

```bash
# get managed node group name (assuming there's only one node group)
GROUP_NAME=$(aws eks list-nodegroups --cluster-name $EKS_CLUSTER_NAME \
  --query nodegroups --out text)
# fetch role arn given node group name
ROLE_ARN=$(aws eks describe-nodegroup --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $GROUP_NAME --query nodegroup.nodeRole --out text)
# extract just the name part of role arn
ROLE_NAME=${ROLE_ARN##*/}
```

:warning: **WARNING**: This will assign allow read-write access to all pods running on the same node pool, not just the ExternalDNS pod(s).

#### Create IAM user and attach the policy

```bash
# create IAM user
aws iam create-user --user-name "externaldns"

# attach policy arn created earlier to IAM user
aws iam attach-user-policy --user-name "externaldns" --policy-arn $POLICY_ARN
```

#### Create the static credentials

```bash
SECRET_ACCESS_KEY=$(aws iam create-access-key --user-name "externaldns")
cat <<-EOF > ./credentials

[default]
aws_access_key_id = $(echo $SECRET_ACCESS_KEY | jq -r '.AccessKey.AccessKeyId')
aws_secret_access_key = $(echo $SECRET_ACCESS_KEY | jq -r '.AccessKey.SecretAccessKey')
EOF
```

#### Create Kubernetes secret from credentials

```bash
kubectl create secret generic external-dns \
  --namespace ${EXTERNALDNS_NS:-"default"} --from-file ./credentials
```



Follow the steps under [Deploy ExternalDNS](#deploy-externaldns) using either RBAC or non-RBAC.  Make sure to uncomment the section that mounts volumes, so that the credentials can be mounted.

### IAM Roles for Service Accounts

[IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) ([IAM roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)) allows cluster operators to map AWS IAM Roles to Kubernetes Service Accounts.  This essentially allows only ExternalDNS pods to access Route53 without exposing any static credentials.

This is the preferred method as it implements [PoLP](https://csrc.nist.gov/glossary/term/principle_of_least_privilege) ([Principal of Least Privilege](https://csrc.nist.gov/glossary/term/principle_of_least_privilege)).

**IMPORTANT**: This method requires using KSA (Kuberntes service account) and RBAC.

This method requires deploying with RBAC.  See [Manifest (for clusters with RBAC enabled)](#manifest-for-clusters-with-rbac-enabled) when ready to deploy ExternalDNS.

**NOTE**: Similar methods to IRSA on AWS are [kiam](https://github.com/uswitch/kiam), which is in maintenence mode, and has [instructions](https://github.com/uswitch/kiam/blob/HEAD/docs/IAM.md) for creating an IAM role, and also [kube2iam](https://github.com/jtblin/kube2iam).  IRSA is the officially supported method for EKS clusters, and so for non-EKS clusters on AWS, these other tools could be an option.

#### Verify OIDC is supported

```bash
aws eks describe-cluster --name $EKS_CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" --output text
```

#### Associate OIDC to cluster

Configure the cluster with an OIDC provider and add support for [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) ([IAM roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)).

If you used `eksctl` to provision the EKS cluster, you can update it with the following command:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $EKS_CLUSTER_NAME --approve
```

#### Create an IAM role bound to a service account

For the next steps in this process, we will need to associate the `external-dns` service account and a role used to grant access to Route53.  This requires the following steps:

1. Create a role with a trust relationship to the cluster's OIDC provider
2. Attach the `AllowExternalDNSUpdates` policy to the role
3. Create the `external-dns` service account
4. Add annotation to the service account with the role arn

##### Use eksctl with eksctl created EKS cluster

If `eksctl` was used to provision the EKS cluster, you can perform all of these steps with the following command:

```bash
eksctl create iamserviceaccount \
  --cluster $EKS_CLUSTER_NAME \
  --name "external-dns" \
  --namespace ${EXTERNALDNS_NS:-"default"} \
  --attach-policy-arn $POLICY_ARN \
  --approve
```

##### Use aws cli with any EKS cluster

Otherwise, we can do the following steps using `aws` commands (also see [Creating an IAM role and policy for your service account](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html)):

```bash
ACCOUNT_ID=$(aws sts get-caller-identity \
  --query "Account" --output text)
OIDC_PROVIDER=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" --output text | sed -e 's|^https://||')

cat <<-EOF > trust.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_PROVIDER"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "$OIDC_PROVIDER:sub": "system:serviceaccount:${EXTERNALDNS_NS:-"default"}:external-dns",
                    "$OIDC_PROVIDER:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
EOF

IRSA_ROLE="external-dns-irsa-role"
aws iam create-role --role-name $IRSA_ROLE --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name $IRSA_ROLE --policy-arn $POLICY_ARN

ROLE_ARN=$(aws iam get-role --role-name $IRSA_ROLE --query Role.Arn --output text)

# Create service account (skip is already created)
kubectl create serviceaccount "external-dns" --namespace ${EXTERNALDNS_NS:-"default"}

# Add annotation referencing IRSA role
kubectl patch serviceaccount "external-dns" --namespace ${EXTERNALDNS_NS:-"default"} --patch \
 "{\"metadata\": { \"annotations\": { \"eks.amazonaws.com/role-arn\": \"$ROLE_ARN\" }}}"
```


If any part of this step is misconfigured, such as the role with incorrect namespace configured in the trust relationship, annotation pointing the the wrong role, etc., you will see errors like `WebIdentityErr: failed to retrieve credentials`. Check the configuration and make corrections.  

When the service account annotations are updated, then the current running pods will have to be terminated, so that new pod(s) with proper configuration (environment variables) will be created automatically.

When annotation is added to service account, the ExternalDNS pod(s) scheduled will have `AWS_ROLE_ARN`, `AWS_STS_REGIONAL_ENDPOINTS`, and `AWS_WEB_IDENTITY_TOKEN_FILE` environment variables injected automatically.

#### Deploy ExternalDNS using IRSA

Follow the steps under [Manifest (for clusters with RBAC enabled)](#manifest-for-clusters-with-rbac-enabled).  Make sure to comment out the service account section if this has been created already.

If you deployed ExternalDNS before adding the service account annotation and the corresponding role, you will likely see error with `failed to list hosted zones: AccessDenied: User`.  You can delete the current running ExternalDNS pod(s) after updating the annotation, so that new pods scheduled will have appropriate configuration to access Route53.


## Set up a hosted zone

*If you prefer to try-out ExternalDNS in one of the existing hosted-zones you can skip this step*

Create a DNS zone which will contain the managed DNS records.  This tutorial will use the fictional domain of `example.com`.

```bash
aws route53 create-hosted-zone --name "example.com." \
  --caller-reference "external-dns-test-$(date +%s)"
```

Make a note of the nameservers that were assigned to your new zone.

```bash
ZONE_ID=$(aws route53 list-hosted-zones-by-name --output json \
  --dns-name "example.com." --query HostedZones[0].Id --out text)

aws route53 list-resource-record-sets --output text \
 --hosted-zone-id $ZONE_ID --query \
 "ResourceRecordSets[?Type == 'NS'].ResourceRecords[*].Value | []" | tr '\t' '\n'
```

This should yield something similar this:

```
ns-695.awsdns-22.net.
ns-1313.awsdns-36.org.
ns-350.awsdns-43.com.
ns-1805.awsdns-33.co.uk.
```

If using your own domain that was registered with a third-party domain registrar, you should point your domain's name servers to the values in the from the list above.  Please consult your registrar's documentation on how to do that.

## Deploy ExternalDNS

Connect your `kubectl` client to the cluster you want to test ExternalDNS with.
Then apply one of the following manifests file to deploy ExternalDNS. You can check if your cluster has RBAC by `kubectl api-versions | grep rbac.authorization.k8s.io`.

For clusters with RBAC enabled, be sure to choose the correct `namespace`.  For this tutorial, the enviornment variable `EXTERNALDNS_NS` will refer to the namespace.  You can set this to a value of your choice:

```bash
export EXTERNALDNS_NS="default" # externaldns, kube-addons, etc

# create namespace if it does not yet exist
kubectl get namespaces | grep -q $EXTERNALDNS_NS || \
  kubectl create namespace $EXTERNALDNS_NS
```


### Manifest (for clusters without RBAC enabled)

Save the following below as `externaldns-no-rbac.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      containers:
        - name: external-dns
          image: k8s.gcr.io/external-dns/external-dns:v0.11.0
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=example.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws
            - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
            - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
            - --registry=txt
            - --txt-owner-id=my-hostedzone-identifier
          env:
            - name: AWS_DEFAULT_REGION
              value: us-east-1 # change to region where EKS is installed
      # # Uncomment below if using static credentials
      #       - name: AWS_SHARED_CREDENTIALS_FILE
      #        value: /.aws/credentials
      #     volumeMounts:
      #       - name: aws-credentials
      #         mountPath: /.aws
      #         readOnly: true
      # volumes:
      #   - name: aws-credentials
      #     secret:
      #       secretName: external-dns
```

When ready you can deploy:

```bash
kubectl create --filename externaldns-no-rbac.yaml \
  --namespace ${EXTERNALDNS_NS:-"default"}
```

### Manifest (for clusters with RBAC enabled)

Save the following below as `externaldns-with-rbac.yaml`.

```yaml
# comment out sa if it was previously created
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods","nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  labels:
    app.kubernetes.io/name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: default # change to desired namespace: externaldns, kube-addons
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: k8s.gcr.io/external-dns/external-dns:v0.11.0
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=example.com # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws
            - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
            - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
            - --registry=txt
            - --txt-owner-id=external-dns
          env:
            - name: AWS_DEFAULT_REGION
              value: us-east-1 # change to region where EKS is installed
     # # Uncommend below if using static credentials
     #        - name: AWS_SHARED_CREDENTIALS_FILE
     #          value: /.aws/credentials
     #      volumeMounts:
     #        - name: aws-credentials
     #          mountPath: /.aws
     #          readOnly: true
     #  volumes:
     #    - name: aws-credentials
     #      secret:
     #        secretName: external-dns
```

When ready deploy:

```bash
kubectl create --filename externaldns-with-rbac.yaml \
  --namespace ${EXTERNALDNS_NS:-"default"}
```

## Arguments

This list is not the full list, but a few arguments that where chosen.

### aws-zone-type

`aws-zone-type` allows filtering for private and public zones

## Annotations

Annotations which are specific to AWS.

### alias

`external-dns.alpha.kubernetes.io/alias` if set to `true` on an ingress, it will create an ALIAS record when the target is an ALIAS as well. To make the target an alias, the ingress needs to be configured correctly as described in [the docs](./nginx-ingress.md#with-a-separate-tcp-load-balancer). In particular, the argument `--publish-service=default/nginx-ingress-controller` has to be set on the `nginx-ingress-controller` container. If one uses the `nginx-ingress` Helm chart, this flag can be set with the `controller.publishService.enabled` configuration option.

## Verify ExternalDNS works (Service example)

Create the following sample application to test that ExternalDNS works.

> For services ExternalDNS will look for the annotation `external-dns.alpha.kubernetes.io/hostname` on the service and use the corresponding value.

> If you want to give multiple names to service, you can set it to external-dns.alpha.kubernetes.io/hostname with a comma `,` separator.

For this verification phase, you can use default or another namespace for the nginx demo, for example:

```bash
NGINXDEMO_NS="nginx"
kubectl get namespaces | grep -q $NGINXDEMO_NS || kubectl create namespace $NGINXDEMO_NS
```

Save the following manifest below as `nginx.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.example.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
```

Deploy the nginx deployment and service with:

```bash
kubectl create --filename nginx.yaml --namespace ${NGINXDEMO_NS:-"default"}
```

Verify that the load balancer was allocated with:

```bash
kubectl get service nginx --namespace ${NGINXDEMO_NS:-"default"}
```

This should show something like:

```bash
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP                                                                   PORT(S)        AGE
nginx   LoadBalancer   10.100.47.41   ae11c2360188411e7951602725593fd1-1224345803.eu-central-1.elb.amazonaws.com.   80:32749/TCP   12m
```

After roughly two minutes check that a corresponding DNS record for your service that was created.

```bash
ZONE_ID=my-route53-zone-id
aws route53 list-resource-record-sets --output json --hosted-zone-id $ZONE_ID \
  --query "ResourceRecordSets[?Name == 'nginx.kubeshore.com.']|[?Type == 'A']"
```

This should show something like:

```json
[
    {
        "Name": "nginx.example.com.",
        "Type": "A",
        "AliasTarget": {
            "HostedZoneId": "ZEWFWZ4R16P7IB",
            "DNSName": "ae11c2360188411e7951602725593fd1-1224345803.eu-central-1.elb.amazonaws.com.",
            "EvaluateTargetHealth": true
        }
    }
]
```

You can also fetch the corresponding text records:

```bash
aws route53 list-resource-record-sets --output json --hosted-zone-id $ZONE_ID \
  --query "ResourceRecordSets[?Name == 'nginx.example.com.']|[?Type == 'TXT']"
```

This will show something like:

```json
[
    {
        "Name": "nginx.example.com.",
        "Type": "TXT",
        "TTL": 300,
        "ResourceRecords": [
            {
                "Value": "\"heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=service/default/nginx\""
            }
        ]
    }
]
```

Note created TXT record alongside ALIAS record. TXT record signifies that the corresponding ALIAS record is managed by ExternalDNS. This makes ExternalDNS safe for running in environments where there are other records managed via other means.

For more information about ALIAS record, see [Choosing between alias and non-alias records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html).

Let's check that we can resolve this DNS name. We'll ask the nameservers assigned to your zone first.

```bash
dig +short @ns-5514.awsdns-53.org. nginx.example.com.
```

This should return 1+ IP addresses that correspond to the ELB FQDN, i.e. `ae11c2360188411e7951602725593fd1-1224345803.eu-central-1.elb.amazonaws.com.`.

Next try the public nameservers configured by DNS client on your system:

```bash
dig +short nginx.example.com.
```

If you hooked up your DNS zone with its parent zone correctly you can use `curl` to access your site.

```bash
curl nginx.example.com.
```

This should show something like:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</head>
<body>
<h1>Welcome to nginx!</h1>
...
</body>
</html>
```

## Verify ExternalDNS works (Ingress example)

With the previous `deployment` and `service` objects deployed, we can add an `ingress` object and configure a FQDN value for the `host` key.  The ingress controller will match incoming HTTP traffic, and route it to the appropriate backend service based on the `host` key.

> For ingress objects ExternalDNS will create a DNS record based on the host specified for the ingress object.

For this tutorial, we have two endpoints, the service with `LoadBalancer` type and an ingress.  For practical purposes, if an ingress is used, the service type can be changed to `ClusterIP` as two endpoints are unecessary in this scenario.

**IMPORTANT**: This requires that an ingress controller has been installed in your Kubernetes cluster.  EKS does not come with an ingress controller by default.  A popular ingress controller is [ingress-nginx](https://github.com/kubernetes/ingress-nginx/), which can be installed by a [helm chart](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx) or by [manifests](https://kubernetes.github.io/ingress-nginx/deploy/#aws).

Create an ingress resource manifest file named `ingress.yaml` with the contents below:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: "nginx" # use the one that corresponds to your ingress controller.
spec:
  rules:
    - host: server.example.com
      http:
        paths:
          - backend:
              service:
                name: nginx
                port:
                  number: 80
            path: /
            pathType: Prefix
```

When ready, you can deploy this with:

```bash
kubectl create --filename ingress.yaml --namespace ${NGINXDEMO_NS:-"default"}
```

Watch the status of the ingress until the ADDRESS field is populated.

```bash
kubectl get ingress --watch --namespace ${NGINXDEMO_NS:-"default"}
```

You should see something like this:

```
NAME    CLASS    HOSTS                ADDRESS   PORTS   AGE
nginx   <none>   server.example.com             80      47s
nginx   <none>   server.example.com   ae11c2360188411e7951602725593fd1-1224345803.eu-central-1.elb.amazonaws.com.   80      54s
```
For the ingress test, run through similar checks, but using domain name used for the ingress:

```bash
# check records on route53
aws route53 list-resource-record-sets --output json --hosted-zone-id $ZONE_ID \
  --query "ResourceRecordSets[?Name == 'server.example.com.']"

# query using a route53 name server
dig +short @ns-5514.awsdns-53.org. server.example.com.
# query using the default name server
dig +short server.example.com.

# connect to the nginx web server through the ingress
curl server.example.com.
```

## More service annotation options

### Custom TTL

The default DNS record TTL (Time-To-Live) is 300 seconds. You can customize this value by setting the annotation `external-dns.alpha.kubernetes.io/ttl`.
e.g., modify the service manifest YAML file above:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.example.com
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
    ...
```

This will set the DNS record's TTL to 60 seconds.

### Routing policies

Route53 offers [different routing policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html). The routing policy for a record can be controlled with the following annotations:

* `external-dns.alpha.kubernetes.io/set-identifier`: this **needs** to be set to use any of the following routing policies

For any given DNS name, only **one** of the following routing policies can be used:

* Weighted records: `external-dns.alpha.kubernetes.io/aws-weight`
* Latency-based routing: `external-dns.alpha.kubernetes.io/aws-region`
* Failover:`external-dns.alpha.kubernetes.io/aws-failover`
* Geolocation-based routing:
  * `external-dns.alpha.kubernetes.io/aws-geolocation-continent-code`
  * `external-dns.alpha.kubernetes.io/aws-geolocation-country-code`
  * `external-dns.alpha.kubernetes.io/aws-geolocation-subdivision-code`
* Multi-value answer:`external-dns.alpha.kubernetes.io/aws-multi-value-answer`

## Clean up

Make sure to delete all Service objects before terminating the cluster so all load balancers get cleaned up correctly.

```bash
kubectl delete service nginx
```

**IMPORTANT** If you attached a policy to the Node IAM Role, then you will want to detach this before deleting the EKS cluster.  Otherwise, the role resource will be locked, and the cluster cannot be deleted, especially if it was provisioned by automation like `terraform` or `eksctl`.

```bash
aws iam detach-role-policy --role-name $NODE_ROLE_NAME --policy-arn $POLICY_ARN
```

If the cluster was provisioned using `eksctl`, you can delete the cluster with:

```bash
eksctl delete cluster --name $EKS_CLUSTER_NAME --region $EKS_CLUSTER_REGION --wait
```

Give ExternalDNS some time to clean up the DNS records for you. Then delete the hosted zone if you created one for the testing purpose.

```bash
aws route53 delete-hosted-zone --id $NODE_ID # e.g /hostedzone/ZEWFWZ4R16P7IB
```

If IAM user credentials were used, you can remove the user with:

```bash
aws iam detach-user-policy --user-name "externaldns" --policy-arn $POLICY_ARN
aws iam delete-user --user-name "externaldns"
```

If IRSA was used, you can remove the IRSA role with:

```bash
aws iam detach-role-policy --role-name $IRSA_ROLE --policy-arn $POLICY_ARN
aws iam delete-role --role-name $IRSA_ROLE
```

Delete any unneeded policies:

```bash
aws iam delete-policy --policy-arn $POLICY_ARN
```

## Throttling

Route53 has a [5 API requests per second per account hard quota](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html#limits-api-requests-route-53).
Running several fast polling ExternalDNS instances in a given account can easily hit that limit. Some ways to reduce the request rate include:
* Reduce the polling loop's synchronization interval at the possible cost of slower change propagation (but see `--events` below to reduce the impact).
  * `--interval=5m` (default `1m`)
* Trigger the polling loop on changes to K8s objects, rather than only at `interval`, to have responsive updates with long poll intervals
  * `--events`
* Limit the [sources watched](https://github.com/kubernetes-sigs/external-dns/blob/master/pkg/apis/externaldns/types.go#L364) when the `--events` flag is specified to specific types, namespaces, labels, or annotations
  * `--source=ingress --source=service` - specify multiple times for multiple sources
  * `--namespace=my-app`
  * `--label-filter=app in (my-app)`
  * `--annotation-filter=kubernetes.io/ingress.class in (nginx-external)` - note that this filter would apply to services too..
* Limit services watched by type (not applicable to ingress or other types)
  * `--service-type-filter=LoadBalancer` default `all`
* Limit the hosted zones considered
  * `--zone-id-filter=ABCDEF12345678` - specify multiple times if needed
  * `--domain-filter=example.com` by domain suffix - specify multiple times if needed
  * `--regex-domain-filter=example*` by domain suffix but as a regex - overrides domain-filter
  * `--exclude-domains=ignore.this.example.com` to exclude a domain or subdomain
  * `--regex-domain-exclusion=ignore*` subtracts it's matches from `regex-domain-filter`'s matches
  * `--aws-zone-type=public` only sync zones of this type `[public|private]`
  * `--aws-zone-tags=owner=k8s` only sync zones with this tag
* If the list of zones managed by ExternalDNS doesn't change frequently, cache it by setting a TTL.
  * `--aws-zones-cache-duration=3h` (default `0` - disabled)
* Increase the number of changes applied to Route53 in each batch
  * `--aws-batch-change-size=4000` (default `1000`)
* Increase the interval between changes
  * `--aws-batch-change-interval=10s` (default `1s`)
* Introducing some jitter to the pod initialization, so that when multiple instances of ExternalDNS are updated at the same time they do not make their requests on the same second.

A simple way to implement randomised startup is with an init container:

```
...
    spec:
      initContainers:
      - name: init-jitter
        image: k8s.gcr.io/external-dns/external-dns:v0.7.6
        command:
        - /bin/sh
        - -c
        - 'FOR=$((RANDOM % 10))s;echo "Sleeping for $FOR";sleep $FOR'
      containers:
...
```

### EKS

An effective starting point for EKS with an ingress controller might look like:

```bash
--interval=5m
--events
--source=ingress
--domain-filter=example.com
--aws-zones-cache-duration=1h
```
