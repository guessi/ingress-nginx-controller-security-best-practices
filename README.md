# Nginx Ingress Controller With Security Protections

Sample script for deployment [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/) in Kubernetes cluster with security protections

## Disclaimer

!!! This is demo purpose scripts, review changes before apply, **DO NOT** apply to production directly !!!

## Prerequisites

- Kubernetes 1.9.x - 1.18.x
- Kubernetes Helm 2.10.x - 3.2.x
- Kubernetes Command Line Tool - [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Why I Create this Repository?

Tons of sample scripts for nginx-ingress-controller, but few of them were security by default.

## Let's Get Started

### Get Helm prepared, and don't forget to check your helm version

    $ helm version

### Ensure helm-repo is up to date

    $ helm repo update

### Install Nginx Ingress Controller

for helm 3.x

    $ helm install nginx-ingress \
        stable/nginx-ingress \
        --namespace kube-system \
        --set rbac.create=true \
        --set controller.kind=DaemonSet

for helm 2.x

    $ helm install --name nginx-ingress \
        stable/nginx-ingress \
        --namespace kube-system \
        --set rbac.create=true \
        --set controller.kind=DaemonSet

**NOTE** it is important to set `rbac.create=true`, and `controller.kind=DaemonSet`

### Deploy

Deploy sample scripts via `kubectl apply`

    $ kubectl apply \
        -f basic-auth.yaml \
        -f backend1.yaml \
        -f backend2.yaml \
        -f ingress.yaml

Check deployment status

    $ kubectl get ingress,service,deployment

    NAME                               HOSTS   ADDRESS                PORTS   AGE
    ingress.extensions/ingress-entry   *       34.56.78.90,34.56...   80      2m

    NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    service/backend1     ClusterIP   10.21.251.1     <none>        8088/TCP   2m
    service/backend2     ClusterIP   10.21.252.2     <none>        8088/TCP   2m
    service/basic-auth   ClusterIP   10.21.253.3     <none>        80/TCP     2m
    service/kubernetes   ClusterIP   10.21.240.1     <none>        443/TCP    5m

    NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.extensions/backend-deployment1   2/2     2            2           2m
    deployment.extensions/backend-deployment2   2/2     2            2           2m
    deployment.extensions/basic-auth            2/2     2            2           2m

### Verification

Before next step, please wait until `EXTERNAL-IP` for `nginx-ingress-controller` being provisioned

    $ kubectl get -n kube-system svc nginx-ingress-controller
    NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    nginx-ingress-controller   LoadBalancer   10.21.251.1     <pending>     80:30625/TCP,443:32598/TCP   51s

Let's check the response for the `/v1` endpoint

    $ curl -i -k -u 'user:mysecretpassword' "https://${NGINX_INGRESS_EXTERNAL_IPADDR}/v1"

        HTTP/1.1 200 OK
        Date: Sat, 01 Jun 2019 12:51:25 GMT
        Content-Type: text/plain; charset=utf-8
        Content-Length: 34
        Connection: keep-alive
        X-App-Name: http-echo
        X-App-Version: 0.2.3
        Strict-Transport-Security: max-age=15724800; includeSubDomains

        "this page is served by backend1"

Let's check the response for the `/v2` endpoint

    $ curl -i -k -u 'user:mysecretpassword' "https://${NGINX_INGRESS_EXTERNAL_IPADDR}/v2"

        HTTP/1.1 200 OK
        Date: Sat, 01 Jun 2019 12:51:27 GMT
        Content-Type: text/plain; charset=utf-8
        Content-Length: 34
        Connection: keep-alive
        X-App-Name: http-echo
        X-App-Version: 0.2.3
        Strict-Transport-Security: max-age=15724800; includeSubDomains

        "this page is served by backend2"

Notice that response header for the http requests:

- `/v1` and `/v2` are routed to different backend services
- `/others` is not defined in nginx-ingress-controller
- Nginx version is not exposed
- Server information is hidden
- Protected by [ModSecurity](https://modsecurity.org/)
- Protected by Basic DoS Protection

### Service without Nginx Ingress Controller

Let's check the response for the `/others` endpoint

    $ curl -i -k -u "https://${NGINX_INGRESS_EXTERNAL_IPADDR}/others"

        HTTP/1.1 404 Not Found
        Server: nginx/1.15.10
        Date: Sat, 01 Jun 2019 12:51:36 GMT
        Content-Type: text/plain; charset=utf-8
        Content-Length: 21
        Connection: keep-alive
        Strict-Transport-Security: max-age=15724800; includeSubDomains

        default backend - 404

Notice that response header for the http requests:

- Server engine info exposed: Nginx
- Nginx version exposed `nginx/1.15.10`

With server side information exposed, attacker could target vulnerabilities for specific to `nginx`, and specific version `nginx/1.15.10`

## Cleanup

Cleanup sample scripts via `kubectl delete`

    $ kubectl delete \
        -f basic-auth.yaml \
        -f backend1.yaml \
        -f backend2.yaml \
        -f ingress.yaml

Cleanup Nginx Ingress Controller

for helm 2.x

    $ helm del --purge nginx-ingress --namespace kube-system

for helm 3.x

    $ helm del nginx-ingress --namespace kube-system

# Reference

- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Nginx Full Configurations Example](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [ModSecurity Web Application Firewall](https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/)
- [Role-Based Access Control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

# License

[GPLv2](LICENSE)
