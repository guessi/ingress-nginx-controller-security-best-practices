# Ingress-NGINX Controller for Kubernetes with Security Best Practices

Sample deployment of [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/) for Kubernetes with security best practices.

## Disclaimer

> :warning: This is demo purpose scripts, review changes before apply, **DO NOT** apply to production directly :warning:

## Prerequisites

- Kubernetes 1.22+
- Kubernetes CLI 1.22+
- Kubernetes Helm 3.10+

## Why I Create this Repository?

Tons of sample scripts for Ingress-Nginx Controller, but few of them were security by default.

## Let's Get Started

### Get Helm prepared, and don't forget to check your helm version

    $ helm version --short
    v3.11.1+g293b50c

### Ensure helm-repo is up to date

    $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    "ingress-nginx" has been added to your repositories

    $ helm repo update ingress-nginx
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "ingress-nginx" chart repository
    Update Complete. ⎈Happy Helming!⎈

### Install Nginx Ingress Controller

    $ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --values values.yaml --wait
    Release "ingress-nginx" does not exist. Installing it now.
    NAME: ingress-nginx
    LAST DEPLOYED: Thu Feb 16 13:16:10 2023
    NAMESPACE: ingress-nginx
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None

### Verify Installation


    $ helm list --filter ingress-nginx --namespace ingress-nginx
    NAME         	NAMESPACE    	REVISION	UPDATED                             	STATUS  	CHART              	APP VERSION
    ingress-nginx	ingress-nginx	1       	2023-02-16 13:16:10.630941 +0800 CST	deployed	ingress-nginx-4.5.2	1.6.4

    $ kubectl get services ingress-nginx-controller  --namespace ingress-nginx
    NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP                         PORT(S)                      AGE
    ingress-nginx-controller   LoadBalancer   10.100.247.220   XXXXX.elb.us-east-1.amazonaws.com   80:31760/TCP,443:31044/TCP   95s

### Detect Installed Version

    $ POD_NAME=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

    $ echo ${POD_NAME}
    ingress-nginx-controller-89758f7c6-hmk86

    $ kubectl -n ingress-nginx exec -it ${POD_NAME} -- /nginx-ingress-controller --version
    -------------------------------------------------------------------------------
    NGINX Ingress controller
      Release:       v1.6.4
      Build:         69e8833858fb6bda12a44990f1d5eaa7b13f4b75
      Repository:    https://github.com/kubernetes/ingress-nginx
      nginx version: nginx/1.21.6
    -------------------------------------------------------------------------------
### Deploy

Deploy sample scripts via `kubectl apply`

    $ kubectl apply -f ./deployment.yaml -f ./service.yaml -f ./ingress.yaml
    deployment.apps/demo-basic-auth created
    deployment.apps/demo-backend created
    service/demo-basic-auth created
    service/demo-backend created
    ingress.networking.k8s.io/demo-ingress created

Check deployment status

    $ kubectl get ingress,service,deployment
    NAME                                     CLASS   HOSTS   ADDRESS                             PORTS   AGE
    ingress.networking.k8s.io/demo-ingress   nginx   *       XXXXX.elb.us-east-1.amazonaws.com   80      20s

    NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    service/demo-backend      ClusterIP   10.100.11.243   <none>        8088/TCP   20s
    service/demo-basic-auth   ClusterIP   10.100.226.9    <none>        80/TCP     21s
    service/kubernetes        ClusterIP   10.100.0.1      <none>        443/TCP    4d14h

    NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/demo-backend      2/2     2            2           21s
    deployment.apps/demo-basic-auth   2/2     2            2           22s

### Verification

Expose backend service entries directly with port-forward

    $ kubectl port-forward service/demo-backend 18088:8088
    Forwarding from 127.0.0.1:18088 -> 5678
    Forwarding from [::1]:18088 -> 5678

Check backend service returns via proxy

    $ curl -i -u 'user:mysecretpassword' "http://localhost:18088/v1"
    HTTP/1.1 200 OK
    X-App-Name: http-echo # <--------------------- Service information exposed.
    X-App-Version: 0.2.3 # <--------------------- Running version information exposed.
    Date: Thu, 16 Feb 2023 05:21:27 GMT
    Content-Length: 34
    Content-Type: text/plain; charset=utf-8

    "hello world"

Wait until ingress endpoint become ready (ADDRESS fieled should show ELB address)

    $ kubectl get ingress
    NAME           CLASS   HOSTS   ADDRESS                             PORTS   AGE
    demo-ingress   nginx   *       XXXXX.elb.us-east-1.amazonaws.com   80      105s

Let's check the responses again with ELB endpoint, HTTPS protocol

    $ curl -i -u 'user:mysecretpassword' "https://${LOAD_BALANCER}/v1" -k
    HTTP/2 200 # <--------------------- Serve with HTTP/2.
    date: Thu, 16 Feb 2023 05:22:29 GMT
    content-type: text/plain; charset=utf-8
    content-length: 34
    strict-transport-security: max-age=15724800; includeSubDomains # <--------------------- No sensitive information expose.

    "hello world"

Let's check the responses again with ELB endpoint, HTTP protocol

    $ curl -i -u 'user:mysecretpassword' "http://${LOAD_BALANCER}/v1"
    HTTP/1.1 308 Permanent Redirect # <--------------------- Securely redirect to HTTPS.
    Date: Thu, 16 Feb 2023 05:22:52 GMT
    Content-Type: text/html
    Content-Length: 164
    Connection: keep-alive
    Location: https://${LOAD_BALANCER}/v1 # <--------------------- Securely redirect to HTTPS.

Try to modify `ingress.yaml`, and see what's the difference

In this example, response header for the http requests:

- Nginx version is not exposed
- Server information is hidden
- Protected by [ModSecurity](https://modsecurity.org/)
- Protected by Basic DoS Protection

## Cleanup

Cleanup sample scripts via `kubectl delete`

    $ kubectl delete -f ./deployment.yaml -f ./service.yaml -f ./ingress.yaml

Cleanup Nginx Ingress Controller

    $ helm uninstall ingress-nginx --namespace ingress-nginx

# Reference

- [Nginx-Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Nginx Full Configurations Example](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [ModSecurity Web Application Firewall](https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/)
- [Role-Based Access Control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

# License

[GPLv2](LICENSE)
