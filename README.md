# Kubernetes(Bare metal/EC2 Instance) Ingress - Letz Encrypt (Auto Renew certificate)
# Deploying the Nginx Ingress  in On prem kubernetes for baremetal and EC2 instance(Standalone) with Letz Encrypt(Auto Renew certificate) . 
- [Introduction](#Introduction)
- [Pre-requisites](#pre-requisites)
- [Installation and configuration](#Installation-and-configuration)
- [Conclusion](#Conclusion)

# Introduction
Kubernetes Ingresses allow you to flexibly route traffic from outside your Kubernetes cluster to Services inside of your cluster. This is accomplished using Ingress Resources, which define rules for routing HTTP and HTTPS traffic to Kubernetes Services, and Ingress Controllers, which implement the rules by load balancing traffic and routing it to the appropriate backend Services.

In this guide, we’ll set up the Kubernetes-maintained Nginx Ingress Controller(hostNetwork configuration), and create some Ingress Resources to route traffic to several dummy backend services. Once we’ve set up the Ingress, we’ll install cert-manager into our cluster to manage and provision TLS certificates for encrypting HTTP traffic to the Ingress.

This will make sure zero cost for your certificate and less manintanice

# Pre-requisites
Before we get started installing the Ingress controller
* Ensure you have Kubernetes 1.15+ cluster on baremetal or in ec2 instance.
* The wget command-line utility installed on your local machine. You can install wget using the package manager built into your operating system.


# Installation and configuration
Clone the project locally to your linux machine.

## Step 1 Setting Up Dummy Backend Services(Change it according to your service)
We’ll first create and roll out dummy echo Services to which we’ll route external traffic using the Ingress.
```
kubectl apply -f http_ingress.yaml
kubectl get svc echo1
```

Verify that the Service started correctly by confirming that it has a ClusterIP, the internal IP on which the Service is exposed:
```
Output
service/echo1 created
deployment.apps/echo1 created
```
Now that our dummy echo web services are up and running, we can move on to rolling out the Nginx Ingress Controller.

## Step 2 - Setting Up the Kubernetes Nginx Ingress Controller with HostNetwork
In this step, we’ll roll out [v0.43.0](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.43.0/deploy/static/provider/baremetal/deploy.yaml) of the Kubernetes-maintained Nginx Ingress Controller. Note that there are several Nginx Ingress Controllers; the Kubernetes community maintains the one used in this guide and Nginx Inc. maintains kubernetes-ingress. The instructions in this tutorial are based on those from the official Kubernetes Nginx Ingress Controller Installation Guide.

In this ingress controler we added hostnetwork is enabled by default

```
kubectl apply -f ingress_controler.yaml
```

We use apply here so that in the future we can incrementally apply changes to the Ingress Controller objects instead of completely overwriting them. To learn more about apply, consult Managing Resources from the official Kubernetes docs.

You should see the following output:

```
Output
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
```

Confirm that the Ingress Controller Pods have started:

```
kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx --watch
```
```
Output
NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-xtbbt       0/1     Completed   0          32s
ingress-nginx-admission-patch-wb8mk        0/1     Completed   1          32s
ingress-nginx-controller-9b98b6577-xrq4k   1/1     Running     0          40s
```
Hit CTRL+C to return to your prompt.

Now, confirm that the NodePort was successfully created by fetching the Service details with kubectl:

```
kubectl get svc --namespace=ingress-nginx
```
```
Output
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.109.108.65   <none>        80:30228/TCP,443:30028/TCP   65s
ingress-nginx-controller-admission   ClusterIP   10.103.80.152   <none>        443/TCP                      65s
```
## Step 3 - Setting Up the Kubernetes Nginx Ingress Resource(HTTP)
Let’s begin by creating a minimal Ingress Resource to route traffic directed at a given subdomain to a corresponding backend Service.

Here, we’ve specified that we’d like to create an Ingress Resource called sample-ingress, and route traffic based on the Host header. An HTTP request Host header specifies the domain name of the target server. 

Sample for your reference
```
  rules:
    - host: app.sample.com
      http:
        paths:
```
to
```
  rules:
    - host: <your_domain>
      http:
        paths:
```
You can now create the Ingress using kubectl:

```
kubectl apply -f http_ingress.yaml
```
You’ll see the following output confirming the Ingress creation:

```
Output
ingress.networking.k8s.io/sample-ingress created
```

Note: Wait for few mins till the external IP is assigned to ingress, you can check that by below commands
```
kubectl get ingress
```
You’ll see the following output confirming the Ingress created with Nodeport ip:
```
NAME             CLASS    HOSTS            ADDRESS       PORTS   AGE
sample-ingress   <none>   app.sample.com   192.168.0.7   80      5s
```

To test the Ingress, navigate to your DNS management service and create A records for app.sample.com(<your_domain>) pointing to the Node IP(In case of EC2 instance open the 80 Port)
Once you’ve created the necessary app.sample.com(<your_domain>) DNS records, you can test the Ingress Controller and Resource you’ve created using the curl command line utility.

From your local machine, curl the echo1 Service:
```
curl http://app.sample.com
```
You should get the following response from the echo1 service:
```
Output
echo1
```
From your local machine, curl the echo1 Service if domain is not mapped:
```
curl -kv http://localhost -H 'Host: app.sample.com'
```
You should get the following response from the echo1 service:
```
Output
* Rebuilt URL to: localhost/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: app.sample.com
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Sat, 30 Jan 2021 18:13:51 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 6
< Connection: keep-alive
< X-App-Name: http-echo
< X-App-Version: 0.2.3
<
echo1
```

## Step 4 — Installing and Configuring Cert-Manager

In this step, we’ll install v0.16.1 of cert-manager into our cluster. cert-manager is a Kubernetes add-on that provisions TLS certificates from [Let’s Encrypt](https://letsencrypt.org/) and other certificate authorities (CAs) and manages their lifecycles. Certificates can be automatically requested and configured by annotating Ingress Resources, appending a tls section to the Ingress spec, and configuring one or more Issuers or ClusterIssuers to specify your preferred certificate authority. To learn more about Issuer and ClusterIssuer objects, consult the official cert-manager documentation on [Issuers](https://cert-manager.io/docs/concepts/issuer/.

Install cert-manager and its Custom Resource Definitions (CRDs) like Issuers and ClusterIssuers by following the official installation instructions. Note that a namespace called cert-manager will be created into which the cert-manager objects will be created:
```
kubectl apply --validate=false -f cert-manager.yaml
or 
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml
```
You should see the following output:

```
Output
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
. . .
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created

```

To verify our installation, check the cert-manager Namespace for running pods:
```
kubectl get pods --namespace cert-manager
```
```
Output
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-271ad6d964-hb512              1/1     Running   0          20s
cert-manager-cainjector-1eeee9dd7c-f46gf   1/1     Running   0          90s
cert-manager-webhook-589b9w9ddd-wf9l6      1/1     Running   0          98s
```

This indicates that the cert-manager installation succeeded.

Before we begin issuing certificates for our echo1.example.com and echo2.example.com domains, we need to create an Issuer, which specifies the certificate authority from which signed x509 certificates can be obtained. In this guide, we’ll use the Let’s Encrypt certificate authority, which provides free TLS certificates and offers both a staging server for testing your certificate configuration, and a production server for rolling out verifiable TLS certificates.

We then specify an email address to register the certificate, and create a Kubernetes Secret called letsencrypt-prod to store the ACME account’s private key. We also use the HTTP-01 challenge mechanism. To learn more about these parameters, consult the official cert-manager documentation on Issuers.

Note: If you reached this steps make sure your domain is mapped to NodePort otherwise certification will not be issued by letz encrypt and make sure 80 and 443 port is open.

Open a file called prod_issuer.yaml in your favorite editor and edit the email id:

```
  ...
    email: <your_email_address_here>
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
  ...
```
Roll out this Issuer using kubectl:
```
kubectl create -f prod_issuer.yaml
```
You should see the following output:
```
Output
clusterissuer.cert-manager.io/letsencrypt-prod created
```

Now that we’ve created our Let’s Encrypt staging and prod ClusterIssuers, we’re ready to modify the Ingress Resource we created above and enable TLS encryption for the echo1.example.com(<your_domain>) paths.

## Step 5 — Issuing Production Let’s Encrypt Certificates

To issue a staging TLS certificate for our domains, we’ll annotate https_ingress_letzencrypt.yaml with the ClusterIssuer created in Step 4. This will use ingress-shim to automatically create and issue certificates for the domains specified in the Ingress manifest.

Open up echo_ingress.yaml in your favorite editor:

```
...
  - hosts:
    - app.sample.com
    secretName: app-secret
  rules:
    - host: app.sample.com
      http:
...
```
to
```
...
  - hosts:
    - <your_domain>
    secretName: app-secret
  rules:
    - host: <your_domain>
      http:
...
```
We also add a tls block to specify the hosts for which we want to acquire certificates, and specify a secretName. This secret will contain the TLS private key and issued certificate. Be sure to swap out app.sample.com with the domain for which you’ve created DNS records.

When you’re done making changes, save and close the file.

We’ll now push this update to the existing Ingress object using kubectl apply:

```
kubectl apply -f echo_ingress.yaml
```

You should see the following output:

```
Output
ingress.networking.k8s.io/sample-ingress configured
```
You can use kubectl describe to track the state of the Ingress changes you’ve just applied:
```
kubectl describe ingress
```
Once the certificate has been successfully created, you can run a describe on it to further confirm its successful creation:

```
kubectl describe certificate
```
Once you see the following output, the certificate has been issued successfully:

```
Output
  Normal  Issuing    28s                 cert-manager  Issuing certificate as Secret was previously issued by ClusterIssuer.cert-manager.io/letsencrypt-prod
  Normal  Reused     28s                 cert-manager  Reusing private key stored in existing Secret resource "app-secret"
  Normal  Requested  28s                 cert-manager  Created new CertificateRequest resource "app-secret-90agn"
  Normal  Issuing    2s (x2 over 4m52s)  cert-manager  The certificate has been successfully issued
```

We’ll now perform a test using curl to verify that HTTPS is working correctly:
```
curl http://app.sample.com
or 
curl -kv http://localhost -H 'Host: app.sample.com'
```
You should see the following:
```
Output
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx/1.15.9</center>
</body>
</html>
```
This indicates that HTTP requests are being redirected to use HTTPS.

Run curl on https:

```
curl https://app.sample.com
or
curl -kv https://localhost -H 'Host: app.sample.com'
```

You should now see the following output:
```
Output
echo1
```
You can run the previous command with the verbose -v flag to dig deeper into the certificate handshake and to verify the certificate information.

At this point, you’ve successfully configured HTTPS using a Let’s Encrypt certificate for your Nginx Ingress.

# Conclusion
In this guide, you set up an Nginx Ingress to load balance and route external requests to backend Services inside of your On prem/EC2 Kubernetes cluster. 
You also secured the Ingress by installing the cert-manager certificate provisioner and setting up a Let’s Encrypt certificate

There are many alternatives to the Nginx Ingress Controller. To learn more, consult [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) from the official Kubernetes documentation.

