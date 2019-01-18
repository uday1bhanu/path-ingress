# path-ingress
Helm chart for custom path based ingress 

# Pre-requisite:

##### Install chart museum
```
mkdir -p charts
docker run --rm -it \
  -p 8080:8080 \
  -e DEBUG=1 \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v $(pwd)/charts:/charts \
  chartmuseum/chartmuseum:latest
```

##### Configmap
```
{
cat <<EOF | tee pathsmap-config.yaml
apiVersion: v1
data:
  paths_map: |
    paths:
    - path: /login
      backend:
        serviceName: svc1-ui
        servicePort: 80
    - path: /about
      backend:
        serviceName: svc2-ui
        servicePort: 80
kind: ConfigMap
metadata:
  name: pathsmap-config
EOF

kubectl create ns dev
kubectl apply -f pathsmap-config.yaml -n dev
}
```

##### RBAC for helm tiller
```
{
kubectl --namespace dev create serviceaccount tiller
kubectl create rolebinding tiller --clusterrole cluster-admin --serviceaccount=dev:tiller --namespace=dev
helm init --service-account tiller --upgrade --tiller-namespace=dev
}
```

##### Clone the repo
```
git clone https://github.com/uday1bhanu/path-ingress.git
```

# Package chart
```
{
mkdir -p artifacts
helm package path-ingress -d artifacts/
}
```

# Upload to chart museum
```
curl --data-binary "@artifacts/path-ingress-0.1.13.tgz" http://localhost:8080/api/charts
```

# Update the repo
```
helm repo update
```

# Install chart
```
{
pathsmap=`kubectl get configmap -n dev pathsmap-config -o template --template='{{.data.paths_map}}'`
helm install --name=path-ingress --version=0.1.13 --debug --tiller-namespace=dev --namespace=dev chartmuseum/path-ingress --set pathsmap="$pathsmap"
}
```