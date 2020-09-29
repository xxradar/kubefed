# Kubefed quickstart
This quickstarts will setup 3 K8S clusters. I used managed K8S clusters on Digitaloceans (latest version)


### Verify access to all 3 clusters

```
export KUBECONFIG=cluster-merge:test1.yaml:test2.yaml:man1.yaml
```
```
kubectl config get-contexts
```
```
kubectl config use-context do-ams3-demo-cluster-test1
kubectl get no

kubectl config use-context do-ams3-demo-cluster-test2
kubectl get no

kubectl config use-context do-ams3-demo-cluster-man1
kubectl get no
```
Make sure the default context is set towards the management clusters

```
kubectl config use-context do-ams3-demo-cluster-man1
kubectl get no
```
### Install helm
```
curl -LO https://git.io/get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: helm
    name: tiller
  name: tiller-deploy
  namespace: kube-system
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: helm
      name: tiller
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: helm
        name: tiller
    spec:
      automountServiceAccountToken: true
      containers:
        - env:
            - name: TILLER_NAMESPACE
              value: kube-system
            - name: TILLER_HISTORY_MAX
              value: '0'
          image: 'gcr.io/kubernetes-helm/tiller:v2.16.9'
          imagePullPolicy: IfNotPresent
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /liveness
              port: 44135
              scheme: HTTP
            initialDelaySeconds: 1
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          name: tiller
          ports:
            - containerPort: 44134
              name: tiller
              protocol: TCP
            - containerPort: 44135
              name: http
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /readiness
              port: 44135
              scheme: HTTP
            initialDelaySeconds: 1
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: tiller
      serviceAccountName: tiller
      terminationGracePeriodSeconds: 30
EOF
```
```
helm init --service-account tiller
```

### install kubefed
check available versions ...
```
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm search kubefed
```
...and install ...(make sure a recent version is used)
```
helm install  kubefed-charts/kubefed  --name=kubefed  --version=0.3.1 --namespace kube-federation-system --devel --debug
```
... verify for success ...
```
kubectl get pod  -n kube-federation-system
```
### Join the clusters
```
kubefedctl join do-ams3-demo-cluster-test1 \
    --host-cluster-name=do-ams3-demo-cluster-man1 \
    --host-cluster-context=do-ams3-demo-cluster-man1 \
    --cluster-context=do-ams3-demo-cluster-test1


kubefedctl join do-ams3-demo-cluster-test2 \
    --host-cluster-name=do-ams3-demo-cluster-man1 \
    --host-cluster-context=do-ams3-demo-cluster-man1 \
    --cluster-context=do-ams3-demo-cluster-test2

```
```
kubectl -n kube-federation-system get kubefedclusters
```

### Deploy a sample app
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: test-namespace
EOF

cat << EOF | kubectl apply -f -
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: test-namespace
  namespace: test-namespace
spec:
  placement:
    clusters:
    - name: do-ams3-demo-cluster-test1
    - name: do-ams3-demo-cluster-test2
EOF

cat << EOF | kubectl apply -f -
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: test-deployment
  namespace: test-namespace
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
  placement:
    clusters:
    - name: do-ams3-demo-cluster-test1
    - name: do-ams3-demo-cluster-test2
EOF
```
... and verify successful deploypent ...
```
kubectl get deployment -n test-namespace --context do-ams3-demo-cluster-test1
kubectl get deployment -n test-namespace --context do-ams3-demo-cluster-test2
```
