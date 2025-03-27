# Ingress-Nginx Controller with Security Best Practices

Sample deployment of [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/) for Kubernetes with security best practices.

## Disclaimer

> :warning: This is demo purpose scripts, review changes before apply, **DO NOT** apply to production directly :warning:

## Why I Create this Repository?

Tons of sample scripts for Ingress-Nginx Controller, but few of them were security by default.

## Before start

:warning: Keep in mind that [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/) but _**NOT**_ [Nginx-ingress Controller](https://docs.nginx.com/nginx-ingress-controller/) are completely different controllers. :warning:

## Let's Get Started

### Verify you are running with a compatible version of the tools

    $ kubectl version --output json
    {
        "clientVersion": {
            "major": "1",
            "minor": "31", # <----------- client version is compatible with server version.
            ...
        },
        "kustomizeVersion": "v5.5.0",
        "serverVersion": {
            "major": "1",
            "minor": "31", # <----------- server version is compatible with client version.
            ...
        }
    }

    $ helm version --short
    v3.17.2+gcc0bbbd

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
    LAST DEPLOYED: Thu Mar 27 09:00:00 2025
    NAMESPACE: ingress-nginx
    STATUS: deployed
    ...
    The ingress-nginx controller has been installed.

### Verify Installation


    $ helm list --filter ingress-nginx --namespace ingress-nginx
    NAME         	NAMESPACE    	REVISION	UPDATED                             	STATUS  	CHART              	APP VERSION
    ingress-nginx	ingress-nginx	1       	2025-03-27 09:00:00.000000 +0800 CST	deployed	ingress-nginx-4.12.1	1.12.1

    $ kubectl get services ingress-nginx-controller --namespace ingress-nginx
    NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                         PORT(S)                      AGE
    ingress-nginx-controller   LoadBalancer   10.100.57.100   XXXXX.elb.us-east-1.amazonaws.com   80:31928/TCP,443:30836/TCP   2m17s

### Detect Installed Version

    $ POD_NAME=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

    $ echo ${POD_NAME}
    ingress-nginx-controller-7cdcd54cc6-4sw5w

    $ kubectl -n ingress-nginx exec -it ${POD_NAME} -- /nginx-ingress-controller --version
    -------------------------------------------------------------------------------
    NGINX Ingress controller
      Release:       v1.12.1
      Build:         51c2b819690bbf1709b844dbf321a9acf6eda5a7
      Repository:    https://github.com/kubernetes/ingress-nginx
      nginx version: nginx/1.25.5
    -------------------------------------------------------------------------------

### Deploy

Deploy sample scripts via `kubectl apply`

    $ kubectl apply -f ./examples
    deployment.apps/demo-basic-auth created
    deployment.apps/demo-backend created
    service/demo-basic-auth created
    service/demo-backend created
    ingress.networking.k8s.io/demo-ingress created

Check deployment status

    $ kubectl get ingress,service,deployment
    NAME                                     CLASS   HOSTS   ADDRESS                             PORTS   AGE
    ingress.networking.k8s.io/demo-ingress   nginx   *       XXXXX.elb.us-east-1.amazonaws.com   80      24s

    NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    service/demo-backend      ClusterIP   10.100.190.107   <none>        8088/TCP   26s
    service/demo-basic-auth   ClusterIP   10.100.203.205   <none>        80/TCP     25s
    service/kubernetes        ClusterIP   10.100.0.1       <none>        443/TCP    2d13h

    NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/demo-backend      1/1     1            1           26s
    deployment.apps/demo-basic-auth   1/1     1            1           25s

### Verification

Expose backend service entries directly with port-forward

    $ kubectl port-forward service/demo-backend 18088:8088
    Forwarding from 127.0.0.1:18088 -> 5678
    Forwarding from [::1]:18088 -> 5678

Check backend service returns via proxy

    $ curl -i -u 'user:mysecretpassword' "http://localhost:18088/v1"
    HTTP/1.1 200 OK
    X-App-Name: http-echo # <--------------------- Service information exposed.
    X-App-Version: 1.0.0 # <--------------------- Running version information exposed.
    Date: Thu, 27 Mar 2025 01:30:00 GMT
    Content-Length: 14
    Content-Type: text/plain; charset=utf-8

    "hello world"

Wait until ingress endpoint become ready (ADDRESS fieled should show ELB address)

    $ kubectl get ingress
    NAME           CLASS   HOSTS   ADDRESS                             PORTS   AGE
    demo-ingress   nginx   *       XXXXX.elb.us-east-1.amazonaws.com   80      55s

Let's check the responses again with ELB endpoint, HTTPS protocol

    $ curl -i -u 'user:mysecretpassword' "https://${LOAD_BALANCER}/v1" -k
    HTTP/2 200 # <--------------------- Serve with HTTP/2.
    date: Thu, 27 Mar 2025 01:30:00 GMT
    content-type: text/plain; charset=utf-8
    content-length: 14
    strict-transport-security: max-age=15724800; includeSubDomains # <--------------------- No sensitive information expose.

    "hello world"

Let's check the responses again with ELB endpoint, HTTP protocol

    $ curl -i -u 'user:mysecretpassword' "http://${LOAD_BALANCER}/v1"
    HTTP/1.1 308 Permanent Redirect # <--------------------- Securely redirect to HTTPS.
    Date: Thu, 27 Mar 2025 01:30:00 GMT
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

    $ kubectl delete -f ./examples

Cleanup Nginx Ingress Controller

    $ helm uninstall ingress-nginx --namespace ingress-nginx

# Reference

- [Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/)
- [Nginx Full Configurations Example](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [ModSecurity Web Application Firewall](https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/)
- [Role-Based Access Control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

# License

[GPLv2](LICENSE)
