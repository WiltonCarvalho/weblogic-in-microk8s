# Microk8s Single Node
# Add Node IP Address to /etc/hosts
```
NODE_IP=$(ip route get 1 | awk '{print $7;exit}')
grep $(hostname) /etc/hosts || echo "$NODE_IP $(hostname)" >> /etc/hosts
```

# Install Microk8s
```
snap install microk8s --classic --channel=1.24/stable
```
# Container Tools
```
apt update
apt -y install curl zip unzip docker.io skopeo openjdk-11-jdk-headless
```
# Alias
```
snap alias microk8s.kubectl kubectl
snap alias microk8s.kubectl k
snap alias microk8s.ctr ctr
```
# Helm
```
microk8s enable helm3
snap alias microk8s.helm3 helm
helm version
```
# Addons
```
microk8s enable rbac
microk8s enable dns
microk8s enable metrics-server
microk8s.enable ingress
microk8s.enable registry
```

# Test Deploy
```
kubectl create deployment httpd --image=httpd --port=80 \
  --dry-run=client -o yaml > httpd.yaml

kubectl apply -f httpd.yaml

kubectl get pods -o wide

kubectl expose deployment httpd --type=NodePort --port=80 --name=httpd \
  --dry-run=client -o yaml > httpd-svc.yaml

kubectl apply -f httpd-svc.yaml

kubectl get svc -o wide

NODE_PORT=$(kubectl describe service httpd | grep ^NodePort | grep -Eo '[0-9]*')
curl -fsSL localhost:$NODE_PORT
```
# Test Ingress
```
cat <<EOF> test-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  ingressClassName: public
  defaultBackend:
    service:
      name: httpd
      port:
        number: 80
EOF
```
```
kubectl apply -f test-ingress.yaml
```
```
curl http://$NODE_IP
curl https://$NODE_IP -k
```
# Push WebLogic Operator image to Microk8s Registry
```
docker pull ghcr.io/oracle/weblogic-kubernetes-operator:4.0.5

docker tag ghcr.io/oracle/weblogic-kubernetes-operator:4.0.5 \
  localhost:32000/oracle/weblogic-kubernetes-operator:4.0.5

docker push localhost:32000/oracle/weblogic-kubernetes-operator:4.0.5
```

# WebLogic Operator Install
```
kubectl create namespace weblogic-operator-ns
```
```
kubectl create serviceaccount -n weblogic-operator-ns weblogic-operator-sa
```
```
helm repo add weblogic-operator \
  https://oracle.github.io/weblogic-kubernetes-operator/charts --force-update
```
```
helm install weblogic-operator weblogic-operator/weblogic-operator \
  --namespace weblogic-operator-ns \
  --set image=localhost:32000/oracle/weblogic-kubernetes-operator:4.0.5 \
  --set serviceAccount=weblogic-operator-sa \
  --set "enableClusterRoleBinding=true" \
  --set "domainNamespaceSelectionStrategy=LabelSelector" \
  --set "domainNamespaceLabelSelector=weblogic-operator\=enabled"
```
```
kubectl -n weblogic-operator-ns get pod
```

# WebLogic Deploy Tooling
```
curl -fSL# \
  https://github.com/oracle/weblogic-deploy-tooling/releases/latest/download/weblogic-deploy.zip \
  -o weblogic-deploy.zip
```
```
curl -fSL# \
  https://github.com/oracle/weblogic-image-tool/releases/latest/download/imagetool.zip \
  -o imagetool.zip && unzip imagetool.zip && rm imagetool.zip
```
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```
```
./imagetool/bin/imagetool.sh cache deleteEntry --key wdt_latest
```
```
./imagetool/bin/imagetool.sh cache addInstaller \
  --type wdt --version latest --path ./weblogic-deploy.zip
```

# Sample Domain
```
curl -fSL# https://github.com/oracle/weblogic-kubernetes-operator/tarball/main | \
  tar zxvf - -C . \
  --one-top-level=sample-domain1 \
  --wildcards \
  "*kubernetes/samples/quick-start/domain-resource.yaml" \
  --strip-components=4
```
# Quickstart App - /quickstart
```
curl -fSL# https://github.com/oracle/weblogic-kubernetes-operator/tarball/main | \
  tar zxvf - -C . \
  --one-top-level=demo-apps/quickstart \
  --wildcards \
    "*kubernetes/samples/quick-start/archive/wlsdeploy/applications/quickstart" \
    "*kubernetes/samples/quick-start/model.yaml" \
    "*kubernetes/samples/quick-start/model.properties" \
  --strip-components=4
```
# WSL Quickstart Model
```
mv demo-apps/quickstart/model.yaml demo-apps/quickstart/quickstart-model.yaml
mv demo-apps/quickstart/model.properties demo-apps/quickstart/quickstart-model.properties
```
# WSL Quickstart Archive
```
(cd demo-apps/quickstart/archive; zip -r ../quickstart-archive.zip wlsdeploy;)
```
# Create an Oracle Account
- https://login.oracle.com
# Log in the Registry webpage, Search for Weblogic and Accept the License Agreement
- "You must agree to and accept the Oracle Standard Terms and Restrictions prior to downloading from the Oracle Container Registry"
- https://container-registry.oracle.com
# WebLogic Base Image
```
docker login container-registry.oracle.com

docker pull container-registry.oracle.com/middleware/weblogic:14.1.1.0-11-ol8

docker tag \
  container-registry.oracle.com/middleware/weblogic:14.1.1.0-11-ol8 \
  localhost:32000/oracle/weblogic:14.1.1.0-11-ol8

docker push localhost:32000/oracle/weblogic:14.1.1.0-11-ol8
```
# Domain FromModel Single Image
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
./imagetool/bin/imagetool.sh update \
  --tag localhost:32000/quickstart-single-image:v1 \
  --fromImage localhost:32000/oracle/weblogic:14.1.1.0-11-ol8 \
  --wdtModel      demo-apps/quickstart/quickstart-model.yaml \
  --wdtVariables  demo-apps/quickstart/quickstart-model.properties \
  --wdtArchive    demo-apps/quickstart/quickstart-archive.zip \
  --wdtModelOnly \
  --wdtDomainType WLS \
  --chown oracle:root
```

# Domain FromModel Aux Image
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
./imagetool/bin/imagetool.sh createAuxImage \
  --tag localhost:32000/quickstart-aux-image:v1 \
  --wdtModel      demo-apps/quickstart/quickstart-model.yaml \
  --wdtVariables  demo-apps/quickstart/quickstart-model.properties \
  --wdtArchive    demo-apps/quickstart/quickstart-archive.zip \
  --wdtHome       /auxiliary
```
# Push Domain images to Microk8s Registry
```
docker push localhost:32000/quickstart-single-image:v1
docker push localhost:32000/quickstart-aux-image:v1
```
# Edit Domain Resources
```
sed -i \
  '/middleware/ s|image:.*|image: "localhost:32000/oracle/weblogic:14.1.1.0-11-ol8"|g' \
  sample-domain1/domain-resource.yaml

sed -i \
  '/quick-start-aux-image:v1/ s|image:.*|image: "localhost:32000/quickstart-aux-image:v1"|g' \
  sample-domain1/domain-resource.yaml
```
# Prepare to Deploy
```
kubectl create namespace sample-domain1-ns

kubectl label namespace sample-domain1-ns weblogic-operator=enabled

helm upgrade weblogic-operator \
  weblogic-operator/weblogic-operator \
  --namespace weblogic-operator-ns \
  --reuse-values \
  --set "domainNamespaces={sample-domain1-ns}" \
  --wait
```

```
kubectl -n sample-domain1-ns create secret generic \
  sample-domain1-weblogic-credentials \
  --from-literal=username=admin \
  --from-literal=password=weblogic1

kubectl -n sample-domain1-ns \
  label secret sample-domain1-weblogic-credentials \
  weblogic.domainUID=sample-domain1

kubectl -n sample-domain1-ns \
  create secret generic sample-domain1-runtime-encryption-secret \
  --from-literal=password=my_runtime_password

kubectl -n sample-domain1-ns \
  label secret sample-domain1-runtime-encryption-secret \
  weblogic.domainUID=sample-domain1
```
# Deploy Domain with Aux Image
```
kubectl apply -f sample-domain1/domain-resource.yaml
```
```
kubectl -n sample-domain1-ns \
  describe domain sample-domain1

kubectl -n sample-domain1-ns get pods --watch
```
# Admin Server Port Forward
```
kubectl -n sample-domain1-ns \
  port-forward pods/sample-domain1-admin-server 7001:7001 --address 0.0.0.0

google-chrome --incognito http://localhost:7001/console

google-chrome --incognito http://192.168.122.40:7001/console

```
# Quickstart App Test
```
CLUSTER_IP=$(kubectl -n sample-domain1-ns \
  get svc sample-domain1-cluster-cluster-1 -o jsonpath='{.spec.clusterIP}')

curl -fsSL http://$CLUSTER_IP:8001/quickstart
```

# Admin Server Node Port
```
cat <<EOF> sample-domain1/domain-resource-patch.yaml 
spec:
  adminServer:
    adminService:
      channels:
      - channelName: default
        nodePort: 30701
EOF
```
```
kubectl -n sample-domain1-ns patch domain sample-domain1 --type merge \
  --patch-file sample-domain1/domain-resource-patch.yaml

kubectl -n sample-domain1-ns get svc

google-chrome --incognito http://192.168.122.40:30701/console
```

# Console Ingress
```
cat <<'EOF'> sample-domain1/ingress-route-console.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-domain1-ingress-console
  namespace: sample-domain1-ns
spec:
  ingressClassName: public
  rules:
    - http:
        paths:
          - path: /console
            pathType: Prefix
            backend:
              service:
                name: sample-domain1-admin-server
                port:
                  number: 7001
EOF
```
```
kubectl apply -f sample-domain1/ingress-route-console.yaml

kubectl -n sample-domain1-ns get ingress

google-chrome --incognito http://192.168.122.40/console
```
# Quickstart Ingress
```
cat <<'EOF'> sample-domain1/ingress-route-quickstart.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-domain1-ingress-quickstart
  namespace: sample-domain1-ns
spec:
  ingressClassName: public
  rules:
    - http:
        paths:
          - path: /quickstart
            pathType: Prefix
            backend:
              service:
                name: sample-domain1-cluster-cluster-1
                port:
                  number: 8001
EOF
```
```
kubectl apply -f sample-domain1/ingress-route-quickstart.yaml

kubectl -n sample-domain1-ns get ingress

google-chrome --incognito http://192.168.122.40/quickstart
```
# Deploy Domain with Single Image
```
cat <<'EOF'> sample-domain1/domain-resource-patch-single-image.yaml
spec:
  configuration:
    model:
      auxiliaryImages: []
  image: "localhost:32000/quickstart-single-image:v1"
EOF
```
```
kubectl -n sample-domain1-ns patch domain sample-domain1 --type merge \
  --patch-file sample-domain1/domain-resource-patch-single-image.yaml
```
```
kubectl -n sample-domain1-ns \
  describe domain sample-domain1

kubectl -n sample-domain1-ns get pods --watch
```
# Sample App - /sample
```
curl -fSL# https://github.com/oracle/docker-images/tarball/main | \
  tar zxvf - -C . \
  --one-top-level=demo-apps/sample/archive/wlsdeploy/applications \
  --wildcards "*OracleWebLogic/samples/12213-deploy-application/sample" \
  --strip-components=4

rm -rf demo-apps/sample/archive/wlsdeploy/applications/sample/META-INF
```
# WSL Sample Archive
```
(cd demo-apps/sample/archive; zip -r ../sample-archive.zip wlsdeploy;)
```
# WSL Sample Model
```
cp demo-apps/quickstart/quickstart-model.yaml demo-apps/sample/sample-model.yaml
cp demo-apps/quickstart/quickstart-model.properties demo-apps/sample/sample-model.properties

sed -i \
  -e 's|applications/quickstart|applications/sample|g' \
  -e 's|quickstart:|sample:|g' \
  demo-apps/sample/sample-model.yaml
```
# Domain FromModel Aux Sample Image
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
./imagetool/bin/imagetool.sh createAuxImage \
  --tag localhost:32000/sample-aux-image:v1 \
  --wdtModel      demo-apps/sample/sample-model.yaml \
  --wdtVariables  demo-apps/sample/sample-model.properties \
  --wdtArchive    demo-apps/sample/sample-archive.zip \
  --wdtHome       /auxiliary
```
# Push Sample image to Microk8s Registry
```
docker push localhost:32000/sample-aux-image:v1
```
# Add the Sample app to sample-domain1
```
vi sample-domain1/domain-resource.yaml
# Add
#       - image: "localhost:32000/sample-aux-image:v1"
#         sourceWDTInstallHome: None
```
```
kubectl apply -f sample-domain1/domain-resource.yaml

kubectl -n sample-domain1-ns get pod
```
# Sample Ingress
```
cat <<'EOF'> sample-domain1/ingress-route-sample.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-domain1-ingress-sample
  namespace: sample-domain1-ns
spec:
  ingressClassName: public
  rules:
    - http:
        paths:
          - path: /sample
            pathType: Prefix
            backend:
              service:
                name: sample-domain1-cluster-cluster-1
                port:
                  number: 8001
EOF
```
```
kubectl apply -f sample-domain1/ingress-route-sample.yaml

kubectl -n sample-domain1-ns get ingress

google-chrome --incognito http://192.168.122.40/sample
```
# Delete
```
helm uninstall weblogic-operator -n weblogic-operator-ns
kubectl delete namespace weblogic-operator-ns
helm repo remove weblogic-operator
kubectl delete namespace sample-domain1-ns
```