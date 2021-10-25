# Manifest

We'll deploy an echo-server with external DNS, certmanager and an ingress controller.
We'll use the proxy protocol to forward client IP from Octavia loadbalancer to the pod.

- Generate a token on https://manager.infomaniak.com/v3/infomaniak-api
- Edit manifest file to replace `change_domain` domain name with yours. 
- Edit kustomization.yaml and replace `changeme` with the generated token.

Apply the kustomize a first time:

- `kustomize build . | kubectl apply -f -`

Some errors will appears because some CRD are not created yet.

Then wait for certmanager and external-dns to be ready then execute the kustomize and second time.

- `kustomize build . | kubectl apply -f -`

A floating ip should be pending:

```bash
kubectl -ningress-nginx get service
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.20.165   <pending>     80:31683/TCP,443:30852/TCP   77s
```

After 1 minute your floating should be assigned.

```bash
kubectl -ningress-nginx get service
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.20.165   195.15.245.99   80:31683/TCP,443:30852/TCP   3m46s
ingress-nginx-controller-admission   ClusterIP      10.233.18.79    <none>          443/TCP  
```

A floating and a loadbalancer appears in openstack

```bash
openstack float ip list
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| ID                                   | Floating IP Address | Fixed IP Address | Port                                 | Floating Network                     | Project                          |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| 0bb528d4-c29c-426c-ab08-95356900d91b | 195.15.245.99       | 10.11.12.24      | 7dddc5bf-037c-4988-8786-d9ea5d302ff1 | 0f9c3806-bd21-490f-918d-4a6d1c648489 | 3bef6ea32166448f96f188c81e25c4c6 |
| d4ae085a-33c4-43ac-8c53-84b71f061f40 | 195.15.245.220      | 10.11.12.81      | 5a2111ba-687b-4ba1-9d88-2ec86059c93c | 0f9c3806-bd21-490f-918d-4a6d1c648489 | 3bef6ea32166448f96f188c81e25c4c6 |
| e13db372-f2ba-4bf4-9376-9e0ee4309db1 | 195.15.246.232      | 10.11.12.73      | 5ca10481-3169-4faf-9e89-1fe4a2214bd3 | 0f9c3806-bd21-490f-918d-4a6d1c648489 | 3bef6ea32166448f96f188c81e25c4c6 |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
openstack loadbalancer list
+--------------------------------------+-------------------------------------------------------------------+----------------------------------+-------------+---------------------+------------------+----------+
| id                                   | name                                                              | project_id                       | vip_address | provisioning_status | operating_status | provider |
+--------------------------------------+-------------------------------------------------------------------+----------------------------------+-------------+---------------------+------------------+----------+
| de322525-6694-4004-9bf3-51a41cdc4388 | k8s-api                                                           | 3bef6ea32166448f96f188c81e25c4c6 | 10.11.12.81 | ACTIVE              | ONLINE           | amphora  |
| 13c78302-3e5c-4209-9b50-57b94f2915a3 | kube_service_cluster.local_ingress-nginx_ingress-nginx-controller | 3bef6ea32166448f96f188c81e25c4c6 | 10.11.12.24 | ACTIVE              | ONLINE           | octavia  |
+--------------------------------------+-------------------------------------------------------------------+----------------------------------+-------------+---------------------+------------------+----------+
```

Your ingress should be ready too.

```bash
kubectl -ndefault get ingress
NAME         CLASS    HOSTS                  ADDRESS         PORTS     AGE
echoserver   <none>   echoserver.change_domain   195.15.245.99   80, 443   5m55s
```

Your DNS A record should be setted automatically by external-dns and Infomaniak API.

```bash
host echoserver.change_domain
echoserver.change_domain has address 195.15.245.99
```

You can check the status of your Let's encrypt certificate.

```bash
kubectl -ndefault get certificate
NAME         READY   SECRET            AGE
echoserver   True    echoserver-cert   20s
```

Finaly call the service. You should see a valid Lets encrypt certificate, and your real client IP address in the `x-real-ip` field

```bash
curl https://echoserver.change_domain

Hostname: echoserver-5c79dc5747-rfj9f

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=10.233.68.3
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://echoserver.change_domain:8080/

Request Headers:
        accept=*/*
        host=echoserver.change_domain
        user-agent=curl/7.64.1
        x-forwarded-for=10.8.8.21
        x-forwarded-host=echoserver.change_domain
        x-forwarded-port=443
        x-forwarded-proto=https
        x-forwarded-scheme=https
        x-real-ip=x.x.x.x
        x-request-id=2706af401c9380eed3a7d09f40a466a7
        x-scheme=https

Request Body:
        -no body in request-
```
