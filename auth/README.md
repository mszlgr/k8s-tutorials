In vanilla k8s kube-apiserver client-certificates are used to authenticate api calls, namelly CommonName is used to extract user name. For ServiceAccounts token auth (`Bearer:`) is used. 

Kubernetes authorization is based on `Role` objects which is list of resources and verbs that can be executed agains those resoruces. Identities (like users and service accounts) are bind to one or more roles using `Rolebinding` objects.

Let's create cluster and check kube-apiserver parameters:
```
kind create cluster --image=kindest/node:v1.23.4
docker exec -it kind-control-plane bash
ps aux | grep kube-apiserver
```

### Inspecting k8s api certs
Let's connect to ks8 API and check what cert is used on server site for https:
```
openssl s_client -showcerts -connect 127.0.0.1:$(docker port kind-control-plane | cut -d':' -f 2) - get cert line and search for it in master pki dir
grep -r <cert_line> -- /etc/kubernetes/pki
```

### How long is ca cert valid?
```
openssl x509 -in ca.crt -text -noout | grep Not
```

# Create new user by issuing cert
Let's start with coping k8s ca certs:
```
docker cp kind-control-plane:/etc/kubernetes/pki/ca.crt ./ca.crt
docker cp kind-control-plane:/etc/kubernetes/pki/ca.key ./ca.key
```

Next we are issuing cert for user, mind `CN=testuser` - Common Name is used by kube-apiserver to autenticate user:
```
openssl genrsa -out testuser.key 2048
openssl req -new -key testuser.key -out testuser.csr -subj "/CN=testuser"
openssl x509 -req -in testuser.csr  -CA ca.crt -CAkey ca.key -CAcreateserial -out testuser.crt -days 1
```

Make sure that `sha256WithRSAEncryption` signeture algorithm is used. `sha1WithRSAEncryption` may not work with newer k8s versions:
```
test with: openssl x509 -in testuser.crt -text -noout
```

Create role and rolebinding for new user:
```
kubectl create role pod-lister --verb=get --verb=list --resource=pods
kubectl create rolebinding testuser-to-pod-lister --role=pod-lister --user=testuser
```

To avoid messing with current enviroment settings lets switch to new config file. Next lest create new config. Kubernetes config file is rely set of 3 lists of `clusters` (api endpoint + ca cert), `users` (username + auth credentials - certs or oidc configuration) and those are binded using `context's` (binding between cluster and user).
```
echo > ~/.kube/new-config
export KUBECONFIG=~/.kube/new-config

kubectl config set-cluster kind --server=https://127.0.0.1:$(docker port kind-control-plane | cut -d':' -f 2) --certificate-authority=ca.crt
kubectl config set-credentials testuser --client-certificate=testuser.crt --client-key=testuser.key
kubectl config set-context testctx --cluster=kind --user=testuser
kubectl config use-context testctx

KUBEAPI_PORT=$(docker port kind-control-plane | cut -d':' -f 2)
curl --cert ./testuser.crt --key ./testuser.key https://127.0.0.1:$KUBEAPI_PORT/api/v1/namespaces/default/pods -v --cacert ./ca.crt
# OR
kubectl get po
```

More info:
k8s authorization doc: https://kubernetes.io/docs/reference/access-authn-authz/authentication/
All about certs in k8s: https://kubernetes.io/docs/setup/best-practices/certificates/

## ServiceAccounts
ServiceAccount is mapping between service name and secret, services need to refer SA in `Pod`/`Deployment` spec using `serviceAccountName: nginx`
ServiceAccount is bind to one or more `Role` using `RoleBinding`

???:
```
k run --image=nginx nginx # need to add
ke nginx -- curl https://kubernetes.default.svc --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Let's run new pod that is configured to use `nginx` ServiceAccount name, and check if proper secret was mounted for pod and we are able to use this secret as Bearer token to auth agains kube-apiserver:
```
kubectl create serviceaccount nginx
kubectl create rolebinding nginx-to-pod-lister --role=pod-lister --serviceaccount=default:nginx

# this should retun "system:serviceaccount:default:nginx", without configured serviceAccountName - "system:serviceaccount:default:default"
ke nginx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d'.' -f2| base64 -d | jq .sub

TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl https://kubernetes/api/v1/namespaces/default/services --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $TOKEN" 
# this should:
# "services is forbidden: User \"system:serviceaccount:default:nginx\" cannot list resource \"services\" in API group \"\" in the namespace \"default\": RBAC: role.rbac.authorization.k8s.io \"pod-lister\" not found"

curl https://kubernetes/api/v1/namespaces/default/pods --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $TOKEN"
```

## Checking users privileges
To check if given `verb` can be performed on a given `resource` by any user or service account `kubectl auth can-i --as <principal>` can be used (its translates into HTTP header `Impersonate-User:`). Example:
```
kubectl auth can-i list pods --as system:serviceaccount:default:nginx
kubectl auth can-i list pods --as testuser
```

## Open questions
Q: Pods is binded to ServiceAccount in pods yaml - if I am able to create pods I am able to select any SA I want - this is a bit problematic from security point of view, right?
