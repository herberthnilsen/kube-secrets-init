# `DISCLAIMER`

This is a fork of the original repo https://github.com/doitintl/kube-secrets-init that was modified to support Oracle Cloud Vault, all the other features still working and wasn't modified


## Using with Oracle Cloud Vault

You need to install the kube-secret-init as usual and describe below on this file, in this repo there are a `values.yaml` file that has a image path to the container of `kube-secret-init` and the initContainer `secret-init`.

The current value is referencing a public repository on Oracle Cloud, but it can be deleted any time, so you will need to build again the container and publish on some container registry

Using the helm command specified below with the option --values=values.yaml, you will get the image that use the code of this repo.

### Before start to use

You need to setup Instace Principal on your Oracle Cloud environment

First you need to create a Dynamic Group to reference all the VMs that the OKE Cluster has using, please check this documentation and use this Matching Rule below

OCI Documentation [Managing Dynamic Group](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingdynamicgroups.htm)

```sh

instance.compartment.id='ocid1.compartment.oc1..<rest of compartment ocid>

```

After that you need to create a OCI Policy that gives permission to that dynamic group

```sh

Allow dynamic-group <DynamicGroup Name> to manage vaults in compartment id <Compartment OCID that has a instance of OCI Vault>

Allow dynamic-group <DynamicGroup Name> to manage keys in compartment id <Compartment OCID that has a instance of OCI Vault>

Allow dynamic-group <DynamicGroup Name> to manage secret-family in compartment <Compartment OCID that has a instance of OCI Vault>

```


### Integration with Oracle Cloud Vault

To use this solution with Oracle Cloud Vault, you need to identify the environment variable using the prefix `oci:vault:` + OCID of your secret on Oracle Cloud Vault

```sh
# environment variable passed to `secrets-init`
MY_API_KEY=oci:vault:ocid.secret.aaaaaaaaaaa

# environment variable passed to child process, resolved by `secrets-init`
MY_API_KEY=key-123456789
```



## Blog Post

[Kubernetes and Secrets Management In The Cloud](https://blog.doit-intl.com/kubernetes-and-secrets-management-in-cloud-part-2-6c37c1238a87?source=friends_link&sk=58405cbafc191a2d7ea2eabbc9d9553e)

# `kube-secrets-init`

The `kube-secrets-init` is a Kubernetes mutating admission webhook, that mutates any K8s Pod that is using specially prefixed environment variables, directly or from Kubernetes as Secret or ConfigMap.

## `kube-secrets-init` mutation

The `kube-secrets-init` injects a `copy-secrets-init` `initContainer` into a target Pod, mounts `/helper/bin` (default; can be changed with the `volume-path` flag) and copies the [`secrets-init`](https://github.com/doitintl/secrets-init) tool into the mounted volume. It also modifies Pod `entrypoint` to `secrets-init` init system, following original command and arguments, extracted either from Pod specification or from Docker image.

### skip injection

The `kube-secrets-init` can be configured to skip injection for all Pods in the specific Namespace by adding the `admission.secrets-init/ignore` label to the Namespace.

## What `secrets-init` does

`secrets-init` runs as `PID 1`, acting like a simple init system. It launches a single process and then proxies all received signals to a session rooted at that child process.

`secrets-init` also passes almost all environment variables without modification, replacing _secret variables_ with values from secret management services.

### Integration with AWS Secrets Manager

User can put AWS secret ARN as environment variable value. The `secrets-init` will resolve any environment value, using specified ARN, to referenced secret value.

```sh
# environment variable passed to `secrets-init`
MY_DB_PASSWORD=arn:aws:secretsmanager:$AWS_REGION:$AWS_ACCOUNT_ID:secret:mydbpassword-cdma3

# environment variable passed to child process, resolved by `secrets-init`
MY_DB_PASSWORD=very-secret-password
```

### Integration with AWS Systems Manager Parameter Store

It is possible to use AWS Systems Manager Parameter Store to store application parameters and secrets.

User can put AWS Parameter Store ARN as environment variable value. The `secrets-init` will resolve any environment value, using specified ARN, to referenced parameter value.

```sh
# environment variable passed to `secrets-init`
MY_API_KEY=arn:aws:ssm:$AWS_REGION:$AWS_ACCOUNT_ID:parameter/api/key

# environment variable passed to child process, resolved by `secrets-init`
MY_API_KEY=key-123456789
```

### Integration with Google Secret Manager

User can put Google secret name (prefixed with `gcp:secretmanager:`) as environment variable value. The `secrets-init` will resolve any environment value, using specified name, to referenced secret value.

```sh
# environment variable passed to `secrets-init`
MY_DB_PASSWORD=gcp:secretmanager:projects/$PROJECT_ID/secrets/mydbpassword
# OR versioned secret (with version or 'latest')
MY_DB_PASSWORD=gcp:secretmanager:projects/$PROJECT_ID/secrets/mydbpassword/versions/2

# environment variable passed to child process, resolved by `secrets-init`
MY_DB_PASSWORD=very-secret-password
```

### Requirement

#### AWS

In order to resolve AWS secrets from AWS Secrets Manager and Parameter Store, `secrets-init` should run under IAM role that has permission to access desired secrets.

This can be achieved by assigning IAM Role to Kubernetes Pod. It's possible to assign IAM Role to EC2 instance, where container is running, but this option is less secure.

#### Google Cloud

In order to resolve Google secrets from Google Secret Manager, `secrets-init` should run under IAM role that has permission to access desired secrets. For example, you can assign the following 2 predefined Google IAM roles to a Google Service Account: `Secret Manager Viewer` and `Secret Manager Secret Accessor` role.

This can be achieved by assigning IAM Role to Kubernetes Pod with [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity). It's possible to assign IAM Role to GCE instance, where container is running, but this option is less secure.

Uncomment `--provider=google` flag in the [deployment.yaml](https://github.com/doitintl/kube-secrets-init/blob/master/deployment/deployment.yaml) file.

## The `kube-secrets-init` deployment

### Deploy with Helm Chart

Consider using the `kube-secrets-init` Helm Chart, authored and managed by [Márk Sági-Kazár](https://github.com/sagikazarmark).

```sh
helm repo add skm https://charts.sagikazarmark.dev
helm install --generate-name --wait skm/kube-secrets-init
```

Check chart GitHub [repository](https://github.com/sagikazarmark/helm-charts/tree/master/charts/kube-secrets-init)

### Manual Deployment

1. To deploy the `kube-secrets-init` server, we need to create a webhook service and a deployment in our Kubernetes cluster. It’s pretty straightforward, except one thing, which is the server’s TLS configuration. If you’d care to examine the [deployment.yaml](https://github.com/doitintl/kube-secrets-init/blob/master/deployment/deployment.yaml) file, you’ll find that the certificate and corresponding private key files are read from command line arguments, and that the path to these files comes from a volume mount that points to a Kubernetes secret:

```yaml
[...]
      args:
      [...]
      - --tls-cert-file=/etc/webhook/certs/cert.pem
      - --tls-private-key-file=/etc/webhook/certs/key.pem
      volumeMounts:
      - name: webhook-certs
        mountPath: /etc/webhook/certs
        readOnly: true
[...]
   volumes:
   - name: webhook-certs
     secret:
       secretName: secrets-init-webhook-certs
```

The most important thing to remember is to set the corresponding CA certificate later in the webhook configuration, so the `apiserver` will know that it should be accepted. For now, we’ll reuse the script originally written by the Istio team to generate a certificate signing request. Then we’ll send the request to the Kubernetes API, fetch the certificate, and create the required secret from the result.

First, run [webhook-create-signed-cert.sh](https://github.com/doitintl/kube-secrets-init/blob/master/deployment/webhook-create-signed-cert.sh) script and check if the secret holding the certificate and key has been created:

```text
./deployment/webhook-create-signed-cert.sh

creating certs in tmpdir /var/folders/vl/gxsw2kf13jsf7s8xrqzcybb00000gp/T/tmp.xsatrckI71
Generating RSA private key, 2048 bit long modulus
.........................+++
....................+++
e is 65537 (0x10001)
certificatesigningrequest.certificates.k8s.io/secrets-init-webhook-svc.default created
NAME                         AGE   REQUESTOR              CONDITION
secrets-init-webhook-svc.default   1s    alexei@doit-intl.com   Pending
certificatesigningrequest.certificates.k8s.io/secrets-init-webhook-svc.default approved
secret/secrets-init-webhook-certs configured
```

**Note** For the GKE Autopilot, run the [webhook-create-self-signed-cert.sh](https://github.com/doitintl/kube-secrets-init/blob/master/deployment/webhook-create-self-signed-cert.sh) script to generate a self-signed certificate.

Export the CA Bundle as a new environment variable `CA_BUNDLE`:

```sh
export CA_BUNDLE=[output value of the previous script "Encoded CA:"]
```

Once the secret is created, we can create deployment and service. These are standard Kubernetes deployment and service resources. Up until this point we’ve produced nothing but an HTTP server that’s accepting requests through a service on port `443`:

```sh
kubectl create -f deployment/deployment.yaml

kubectl create -f deployment/service.yaml
```

### configure mutating admission webhook

Now that our webhook server is running, it can accept requests from the `apiserver`. However, we should create some configuration resources in Kubernetes first. Let’s start with our validating webhook, then we’ll configure the mutating webhook later. If you take a look at the [webhook configuration](https://github.com/doitintl/kube-secrets-init/blob/master/deployment/mutatingwebhook.yaml), you’ll notice that it contains a placeholder for `CA_BUNDLE`:

```yaml
[...]
      service:
        name: secrets-init-webhook-svc
        namespace: default
        path: "/pods"
      caBundle: ${CA_BUNDLE}
[...]
```

There is a [small script](https://github.com/doitintl/kube-secrets-init/blob/master/deployment/webhook-patch-ca-bundle.sh) that substitutes the CA_BUNDLE placeholder in the configuration with this CA. Run this command before creating the validating webhook configuration:

```sh
cat ./deployment/mutatingwebhook.yaml | ./deployment/webhook-patch-ca-bundle.sh > ./deployment/mutatingwebhook-bundle.yaml
```

Create mutating webhook configuration:

```sh
kubectl create -f deployment/mutatingwebhook-bundle.yaml
```

### configure RBAC for secrets-init-webhook

Create Kubernetes Service Account to be used with `secrets-init-webhook`:

```sh
kubectl create -f deployment/service-account.yaml
```

Define RBAC permission for webhook service account:

```sh
# create a cluster role
kubectl create -f deployment/clusterrole.yaml
# define a cluster role binding
kubectl create -f deployment/clusterrolebinding.yaml
```
