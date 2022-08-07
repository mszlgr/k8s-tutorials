kind create cluster --image=kindest/node:v1.23.4
docker exec -it kind-control-plane bash
ps aux | grep kube-apiserver


* check cert for kube-api
openssl s_client -showcerts -connect 127.0.0.1:$(docker port kind-control-plane | cut -d':' -f 2) - get cert line and search for it in master pki dir
grep -r <cert_line> -- /etc/kubernetes/pki

* how long is ca cert valid?
openssl x509 -in ca.crt -text -noout | grep Not

* copy crt and key
docker cp kind-control-plane:/etc/kubernetes/pki/ca.crt ./ca.crt
docker cp kind-control-plane:/etc/kubernetes/pki/ca.key ./ca.key

*
openssl genrsa -out testuser.key 2048
openssl req -new -key testuser.key -out testuser.csr -subj "/CN=testuser/O=testing"
openssl x509 -req -in testuser.csr  -CA ca.crt -CAkey ca.key -CAcreateserial -out testuser.crt -days 1

# test with: openssl x509 -in testuser.crt -text -noout
# IMPORTANT - Signature Algorithm: sha1WithRSAEncryption will not work, need to be sha256WithRSAEncryption

*
kubectl create role pod-lister --verb=get --verb=list --resource=pods
kubectl create rolebinding testuser-to-pod-lister --role=pod-lister --user=testuser

*
echo > ~/.kube/new-config
export KUBECONFIG=~/.kube/new-config

kubectl config set-cluster kind --server=https://127.0.0.1:$(docker port kind-control-plane | cut -d':' -f 2) --certificate-authority=ca.crt
kubectl config set-credentials testuser --client-certificate=testuser.crt --client-key=testuser.key
kubectl config set-context testctx --cluster=kind --user=testuser
kubectl config use-context testctx

KUBEAPI_PORT=$(docker port kind-control-plane | cut -d':' -f 2)
curl --cert ./testuser.crt --key ./testuser.key https://127.0.0.1:$KUBEAPI_PORT/api/v1/namespaces/default/pods -v --cacert ./ca.crt
OR
kubectl get po

More info:
k8s authorization doc: https://kubernetes.io/docs/reference/access-authn-authz/authentication/
All about certs in k8s: https://kubernetes.io/docs/setup/best-practices/certificates/

---- SA
SA is mapping bettween service name and secret, service need to refer SA in spec like `serviceAccountName: nginx`
SAB is mapping between SA and Role

k run --image=nginx nginx # need to add
ke nginx -- curl https://kubernetes.default.svc -k
# or curl https://kubernetes.default.svc --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt

*
kubectl create serviceaccount nginx
kubectl create rolebinding nginx-to-pod-lister --role=pod-lister --serviceaccount=default:nginx

ke nginx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d'.' -f2| base64 -d


TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl https://kubernetes --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $TOKEN" 
# this should return forbidden: User \"system:serviceaccount:default:default\" cannot get path

curl https://kubernetes/api/v1/namespaces/default/pods --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $TOKEN"
