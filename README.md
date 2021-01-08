# Using a wildcard certificate in AKS
This repository demonstrates how to add Kubernetes Ingress using Nginx to a cluster in Azure Kubernetes Services. Moreover, it covers adding TLS termination to custom domain using Let's Encrypt free services to request a wildcard certificate using cert-manager. Example covers how to complete DNS verification using Azure DNS Zones, however additional  DNS providers are supported (https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers)

# Context
During an app migration project from Azure App Service to AKS, I was asked to find a way to use a wildcard certificate to apps deployed in AKS, the request also required to have a default route to any subdomain to be handled by the same app. I will show the steps needed to accomplish this. For reference the AKS version used is 1.19.3 and the cert-manager version is 1.1

# Environment Setup
You can try this approach in your existing AKS cluster or setup a test environment. The instructions provided were tested on cert-manager v1.1, if you are using an older version you can try uninstalling and install using the instructions below. If you are starting from zero you can build your test environment by doing the following:

1. Get access to an Azure Subscription
2. Go to https://docs.microsoft.com/en-us/learn/modules/aks-workshop/ and deploy a new AKS cluster following the instructions provided there from the Introduction until the end of the exercise "Deploy an ingress for the front end" 

# Locate the service principal of the AKS

The first step is to locate the service principal that the AKS cluster uses so it can be granted access to the Azure DNS Zone that manages the domain you want to ask for a wildcard certificate. If your AKS cluster doesn't use a service principal, then you can proceed to create a new app registration in your Azure Active Directory, obtain the ClientID and generate a secret that will be used later on.

You can check for the service principal used in AKS cluster in your subscription by running

 - az aks list --query "[].{ name: name, servicePrincipalProfile: servicePrincipalProfile }"

# Add the service principal as contributor of your DNS Zone

```bash
# Find out resource group of the DNS Zone
$ az network dns zone list --query "[].{ id: id, name: name}"

# Add the service principal as contributor
$ az role assignment create --assignee <service principal> --role Contributor --scope <dns-zone-id>
```

This can be done using the Portal (Azure DNS Zone / Access Control / Add role contributor)

# Create secret containing the service principal password

Inside the Azure Portal, select Azure Active Directory and click on App registrations, select all applications and paste the service principal used by the AKS cluster or the app manually generated so you can proceed to create a new client secret. Click on +New client secret, provide a description and a expiration option (may be Never) and copy the provided secret so it can be converted into base 64


```bash
# Get the password converted in base 64
$ echo <the-password> | openssl base64
<the-password-in-base64>
```

Then proceed to create the secret in Kubernetes (you can check the folder sample for reference)

```bash
apiVersion: v1
kind: Secret
metadata:
  name: secret-azuredns-config
data:
  # echo <service principal password> | openssl base64
  password: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
```

If you are following the aksworkshop sample, you should save this secret on the same namespace (ratingsapp for that scenario)

```bash
kubectl apply -f secret.yaml --namespace ratingsapp
```


# Add an A record in the Azure DNS Zone

It is recommended that you get the public IP of the service "nginx-ingress-ingress-nginx-controller" and add an A record in the Azure DNS Zone 

```bash
Details:

Name: *
Type: A
Alias record set: No
TTL: (Default) 1 hours
IP Address: The public IP of the nginx ingress controller
```

Also, it is important to delete the lock so delete operations are allowed on the Azure DNS Zone, it is important to delete this lock so the verification can be completed and the certificate is issued properly.

# Install cert-manager v1.1

Note: The following instructions has only been tested on verion 1.1 of cert-manager.

Follow the instructions on https://cert-manager.io/docs/installation/kubernetes/ to install cert-manager v1.1  using Helm and the Option 2: install CRDs as part of the Helm release

```bash
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true

```

Also, don't forget to verify the installation of cert-manager by following https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation


# Create the Issuer using the service principal

The file issuer.yaml can be found in the sample folder, please update the following details:

```bash
e-mail
clientID: Use the service principal or ClientID
subscriptionID: Use the ID of your Azure Subscription
tenantID: Use the ID of the Azure AD of your Subscription
resourceGroupName: Make sure to specify the resource group of the Azure DNS zone you are using
hostedZoneName: use the name of the Azure DNS zone
```

You can find the details on this configuration file at https://cert-manager.io/docs/configuration/acme/dns01/azuredns/

You also can use another DNS provider like Cloud Flare, just follow the instructions at https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/

You can check the DNS providers that can be used in cert-manager at https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers

Finally, apply the issuer file (for this example, we continue to use the raingsapp of the aksworkshop sample)

```bash
kubectl apply -f issuer.yaml --namespace ratingsapp
```

# Request a wildcard certificate for the domain hosted in the DNS zone

The file certificate.yaml can be found in the sample folder, please update the following details:

```bash
common name: update to *.yourcustomdomain.com
dnsNames: update to "*.yourcustomdomain.com" and yourcustomdomain.com
```

Finally, apply the certificate file (for this example, we continue to use the raingsapp of the aksworkshop sample)

```bash
kubectl apply -f certificate.yaml --namespace ratingsapp
```

Watch for the certificate to check when it is ready to use

```bash
kubectl describe cert wildcard --namespace ratingsapp
```

Also, you can check for the events happening in case you need to troubleshoot, just run the following

```bash
kubectl get events -w --namespace ratingsapp
```

# Create the ingress rule

The file ingress.yaml can be found in the sample folder, please update the following details:

```bash
hosts: "*.yourcustomdomain.com"
rules: demo.yourcustomdomain.com, "*.yourcustomdomain.com", yourcustomdomain.com
servicename: Use the service that you want to be exposed on each of the rules, on the * rule you can specify the default service to handle all the non-specific subdomain request
```

Apply the ingress rule and check for the results

```bash
kubectl apply -f ingress.yaml --namespace ratingsapp
```