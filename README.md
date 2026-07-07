Upgrade Steps:

# synchronize secrets
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key --context kind-mycluster -o yaml > mycluster.yaml
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key --context kind-mycluster2 -o yaml >  mycluster2.yaml
kubectl create -f mycluster.yaml --context kind-mycluster2
kubectl create -f mycluster2.yaml --context kind-mycluster

# helm upgrade to set keyrenewperiod=0
helm upgrade --reuse-values smf-sealed-secrets ./sealed-secrets-2.17.0 -n kube-system --set keyrenewperiod=0 --kube-context kind-mycluster
helm upgrade --reuse-values smf-sealed-secrets ./sealed-secrets-2.17.0 -n kube-system --set keyrenewperiod=0 --kube-context kind-mycluster2

# add owner labels to all secrets
kubectl label secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key sealedsecrets.bitnami.com/managed="true" --context kind-mycluster
kubectl label secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key sealedsecrets.bitnami.com/managed="true" --context kind-mycluster2

# Create and apply new argocd key for each partition (1 here)
export PRIVATEKEY="argo-key.key"
export PUBLICKEY="argo-cert.crt"
export NAMESPACE="kube-system"
export SECRETNAME="sealed-secrets-key-argocd"
openssl req -x509 -days 365 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY" --context kind-mycluster
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active --context kind-mycluster
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY" --context kind-mycluster2
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active --context kind-mycluster2

# encrypt all keys with argocd key
mkdir -p sealed-secrets-keys

kubectl get secrets -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' \
| while read -r secret; do
    kubectl get secret "$secret" -n kube-system -o yaml | kubeseal --cert argo-cert.crt -o yaml > "sealed-secrets-keys/${secret}.yaml"
done

# Create new key to be used as current and encrypt with argocd key

export SUFFIX="$(tr -dc 'a-z0-9' </dev/urandom | head -c 5)"
export PRIVATEKEY="sealed-secrets-key${SUFFIX}.key"
export PUBLICKEY="sealed-secrets-key${SUFFIX}.crt"
export NAMESPACE="kube-system"
export SECRETNAME="sealed-secrets-key${SUFFIX}"
openssl req -x509 -days 365 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY" --dry-run=client -o yaml | kubectl label -f - "sealedsecrets.bitnami.com/sealed-secrets-key=active" --local -o yaml | kubeseal --cert argo-cert.crt -o yaml > "sealed-secrets-keys/${SECRETNAME}.yaml"

# Add new keys to git project

mv sealed-secrets-keys/* sealed-secrets-argocd

# git rid of helm

# apply argo application



