# Introduction to cert-manager for Kubernetes

## We need a Kubernetes cluster


## Concepts 

It's important to understand the various concepts and new Kubernetes resources that <br/>
`cert-manager` introduces.

* Issuers [docs](https://cert-manager.io/docs/concepts/issuer/)
* Certificate [docs](https://cert-manager.io/docs/concepts/certificate/)
* CertificateRequests [docs](https://cert-manager.io/docs/concepts/certificaterequest/)
* Orders and Challenges [docs](https://cert-manager.io/docs/concepts/acme-orders-challenges/)

## Installation 

You can find the latest release for `cert-manager` on their [GitHub Releases page](https://github.com/jetstack/cert-manager/) <br/>

For this demo, I will use K8s 1.23 and `cert-manager` [v1.10.0](https://github.com/jetstack/cert-manager/releases/tag/v1.10.0)

```

# get cert-manager 

cd kubernetes/cert-manager/
curl -LO https://github.com/jetstack/cert-manager/releases/download/v1.10.0/cert-manager.yaml

mv cert-manager.yaml cert-manager-1.10.0.yaml

# install cert-manager 

kubectl apply -f cert-manager-1.10.0.yaml
```

## Cert Manager Resources

We can see our components deployed

```
kubectl -n cert-manager get all

NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-86548b886-2b8x7               1/1     Running   0          77s
pod/cert-manager-cainjector-6d59c8d4f7-hrs2v   1/1     Running   0          77s
pod/cert-manager-webhook-578954cdd-tphpj       1/1     Running   0          77s

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.96.87.136   <none>        9402/TCP   77s
service/cert-manager-webhook   ClusterIP   10.104.59.25   <none>        443/TCP    77s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE        
deployment.apps/cert-manager              1/1     1            1           77s
deployment.apps/cert-manager-cainjector   1/1     1            1           77s
deployment.apps/cert-manager-webhook      1/1     1            1           77s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-86548b886               1         1         1       77s
replicaset.apps/cert-manager-cainjector-6d59c8d4f7   1         1         1       77s
replicaset.apps/cert-manager-webhook-578954cdd       1         1         1       77

```

## Create cert-manager IAM user to handle creation of the dns records in route53 

```
 aws iam create-policy --policy-name letsencrypt-wildcard --policy-document file://cert-manager-iam.json
{
    "Policy": {
        "PolicyName": "letsencrypt-wildcard",
        "PolicyId": "ANPAWLKVZU6YWANTOG2J7",
        "Arn": "arn:aws:iam::436655990705:policy/letsencrypt-wildcard",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-11-16T23:25:42+00:00",
        "UpdateDate": "2022-11-16T23:25:42+00:00"
    }
}
```
export variable to get the IAM policy, create group, attach policy to the group

```
LE_POLICY_ARN=$(aws iam list-policies --output json --query 'Policies[*].[PolicyName,Arn]' --output text | grep letsencrypt-wildcard | awk '{print $2}')

aws iam create-group --group-name letsencrypt-wildcard

aws iam attach-group-policy --policy-arn ${LE_POLICY_ARN} --group-name letsencrypt-wildcard

aws iam create-user --user-name letsencrypt-wildcard

aws iam add-user-to-group --user-name letsencrypt-wildcard --group-name letsencrypt-wildcard

aws iam create-access-key --user-name letsencrypt-wildcard # COPY and SAVE the output of this command

```
## Add AccessKeyID to clusterissuer-dns01.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: cggor8@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory  
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-account-key    
    solvers:
    - selector:
        dnsZones:
          - "kubeshore.com"
      dns01:
        route53:
          region: us-east-2
          accessKeyID: <ACCESSKEYID> # Insert Access Key ID from the output of aws iam create-access-key
          secretAccessKeySecretRef:
            name: route53-credentials-secret
            key: secret-access-key
```

## Create kubernetes secret.yaml in cert-manager namespace.

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: cert-manager
  name: route53-credentials-secret
type: Opaque  
stringData:
  secret-access-key: SecretAccessKey # paste from aws iam create-access-key --user-name letsencrypt-wildcard
```

when ready deploy the secret.yaml

```
kubectl apply -f secret.yaml
```


## Setup my DNS

using external-dns

## Create Let's Encrypt Issuer for our cluster

We create a `ClusterIssuer` that allows us to issue certs in any namespace
View all the chain components triggered

```
kubectl get clusterissuer,certificates,certificaterequest,order,challenge

NAME                                                   READY   AGE
clusterissuer.cert-manager.io/letsencrypt-production   True    6m

NAME                                    READY   SECRET      AGE
certificate.cert-manager.io/echo-tls    True    echo-tls    4m43s

NAME                                                 APPROVED   DENIED   READY   ISSUER                   REQUESTOR                                         AGE
certificaterequest.cert-manager.io/echo-tls-zzkld    True                True    letsencrypt-production   system:serviceaccount:cert-manager:cert-manager   4m43s


NAME                                                    STATE   AGE
order.acme.cert-manager.io/echo-tls-zzkld-1733386478    valid   4m43sorder.acme.cert-manager.io/nginx-tls-hsgvl-4245505514   valid   2m40s
```

```
kubectl apply -f clusterissuer-dns01.yaml

# check the issuer
kubectl describe clusterissuer letsencrypt-cluster-issuer


# check the cert has been issued 

kubectl describe certificate name

# TLS Certificate created as a secret
kubectl get secrets
NAME                  TYPE                                  DATA   AGE
example-app-tls       kubernetes.io/tls                     2      84m
```
