# Ingress-NGINX Controller for Kubernetes with Security Best Practices

Sample deployment of [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/) for Kubernetes with security best practices.

## Disclaimer

> :warning: This is demo purpose scripts, review changes before apply, **DO NOT** apply to production directly :warning:

## Prerequisites

- Kubernetes 1.21+
- Kubernetes CLI 1.21+
- Kubernetes Helm 3.9+

## Why I Create this Repository?

Tons of sample scripts for Ingress-Nginx Controller, but few of them were security by default.

## Let's Get Started

### Get Helm prepared, and don't forget to check your helm version

    $ helm version --short
    v3.9.4+gdbc6d8e

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
    LAST DEPLOYED: Mon Sep 19 13:14:25 2022
    NAMESPACE: ingress-nginx
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
### Verify Installation

    $ helm list --filter ingress-nginx --namespace ingress-nginx
    NAME         	NAMESPACE    	REVISION	UPDATED                             	STATUS  	CHART              	APP VERSION
    ingress-nginx	ingress-nginx	1       	2022-09-19 13:14:25.993815 +0800 CST	deployed	ingress-nginx-4.2.5	1.3.1

    $ kubectl get services ingress-nginx-controller  --namespace ingress-nginx
    NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                         PORT(S)                      AGE
    ingress-nginx-controller   LoadBalancer   10.100.49.36    XXXXX.elb.us-east-1.amazonaws.com   80:30547/TCP,443:30772/TCP   2m13s
### Detect Installed Version

    $ POD_NAME=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

    $ echo ${POD_NAME}
    ingress-nginx-controller-55dcf56b68-72m8v

    $ kubectl -n ingress-nginx exec -it ${POD_NAME} -- /nginx-ingress-controller --version
    -------------------------------------------------------------------------------
    NGINX Ingress controller
    Release:       v1.3.1
    Build:         92534fa2ae799b502882c8684db13a25cde68155
    Repository:    https://github.com/kubernetes/ingress-nginx
    nginx version: nginx/1.19.10
    -------------------------------------------------------------------------------

### Deploy

Deploy sample scripts via `kubectl apply`

    $ kubectl apply -f ./deployment.yaml -f ./service.yaml -f ./ingress.yaml
    deployment.apps/demo-basic-auth created
    deployment.apps/demo-backend-1 created
    deployment.apps/demo-backend-2 created
    service/demo-basic-auth created
    service/demo-backend-1 created
    service/demo-backend-2 created
    ingress.networking.k8s.io/demo-ingress created

Check deployment status

    $ kubectl get ingress,service,deployment
    NAME                                     CLASS   HOSTS   ADDRESS                             PORTS   AGE
    ingress.networking.k8s.io/demo-ingress   nginx   *       XXXXX.elb.us-east-1.amazonaws.com   80      89s

    NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    service/demo-backend-1    ClusterIP   10.100.69.221    <none>        8088/TCP   94s
    service/demo-backend-2    ClusterIP   10.100.124.153   <none>        8088/TCP   93s
    service/demo-basic-auth   ClusterIP   10.100.188.204   <none>        80/TCP     96s
    service/kubernetes        ClusterIP   10.100.0.1       <none>        443/TCP    8d

    NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/demo-backend-1     2/2     2            2           101s
    deployment.apps/demo-backend-2     2/2     2            2           99s
    deployment.apps/demo-basic-auth    2/2     2            2           102s
### Verification

Expose backend service entries directly with port-forward

    $ kubectl port-forward service/demo-backend-1 18088:8088
    Forwarding from 127.0.0.1:18088 -> 5678
    Forwarding from [::1]:18088 -> 5678

    $ kubectl port-forward service/demo-backend-2 28088:8088
    Forwarding from 127.0.0.1:28088 -> 5678
    Forwarding from [::1]:28088 -> 5678

Check backend service returns via proxy

    $ curl -i -u 'user:mysecretpassword' "http://localhost:18088/v1"
    HTTP/1.1 200 OK
    X-App-Name: http-echo # <--------------------- Service information exposed.
    X-App-Version: 0.2.3 # <--------------------- Running version information exposed.
    Date: Mon, 19 Sep 2022 05:23:11 GMT
    Content-Length: 34
    Content-Type: text/plain; charset=utf-8

    "this page is served by backend1"

    $ curl -i -u 'user:mysecretpassword' "http://localhost:28088/v2"
    HTTP/1.1 200 OK
    X-App-Name: http-echo # <--------------------- Service information exposed.
    X-App-Version: 0.2.3 # <--------------------- Running version information exposed.
    Date: Mon, 19 Sep 2022 05:23:32 GMT
    Content-Length: 34
    Content-Type: text/plain; charset=utf-8

    "this page is served by backend2"

Wait until ingress endpoint become ready (ADDRESS fieled should show ELB address)

    $ kubectl get ingress
    NAME           CLASS   HOSTS   ADDRESS                             PORTS   AGE
    demo-ingress   nginx   *       XXXXX.elb.us-east-1.amazonaws.com   80      3m6s

Let's check the responses again with ELB endpoint, HTTPS protocol

    $ curl -i -u 'user:mysecretpassword' "https://${LOAD_BALANCER}/v1" -k
    HTTP/2 200 # <--------------------- Serve with HTTP/2.
    date: Mon, 19 Sep 2022 05:24:32 GMT
    content-type: text/plain; charset=utf-8
    content-length: 34
    strict-transport-security: max-age=15724800; includeSubDomains # <--------------------- No sensitive information expose.

    "this page is served by backend1"

    $ curl -i -u 'user:mysecretpassword' "https://${LOAD_BALANCER}/v2" -k
    HTTP/2 200 # <--------------------- Serve with HTTP/2.
    date: Mon, 19 Sep 2022 05:24:36 GMT
    content-type: text/plain; charset=utf-8
    content-length: 34
    strict-transport-security: max-age=15724800; includeSubDomains # <--------------------- No sensitive information expose.

    "this page is served by backend2"

Let's check the responses again with ELB endpoint, HTTP protocol

    $ curl -i -u 'user:mysecretpassword' "http://XXXXX.elb.us-east-1.amazonaws.com/v1"
    HTTP/1.1 308 Permanent Redirect # <--------------------- Securely redirect to HTTPS.
    Date: Mon, 19 Sep 2022 05:25:23 GMT
    Content-Type: text/html
    Content-Length: 164
    Connection: keep-alive
    Location: https://XXXXX.elb.us-east-1.amazonaws.com/v1 # <--------------------- Securely redirect to HTTPS.

    $ curl -i -u 'user:mysecretpassword' "http://XXXXX.elb.us-east-1.amazonaws.com/v2"
    HTTP/1.1 308 Permanent Redirect # <--------------------- Securely redirect to HTTPS.
    Date: Mon, 19 Sep 2022 05:25:48 GMT
    Content-Type: text/html
    Content-Length: 164
    Connection: keep-alive
    Location: https://XXXXX.elb.us-east-1.amazonaws.com/v2 # <--------------------- Securely redirect to HTTPS.


Try to modify `ingress.yaml`, and see what's the difference

In this example, response header for the http requests:

- `/v1` and `/v2` are routed to different backend services
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
