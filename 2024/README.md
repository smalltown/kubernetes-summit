# Kubernetes Summit 2024

## 基本環境設定

### 安裝 Docker Desktop
用來運行 Container，安裝檔案下載連結：https://www.docker.com/products/docker-desktop

### 安裝 kind
用來運行輕量 K8s Cluster 於本地端，非 macOS 安裝方式請參閱[官方文件](https://kind.sigs.k8s.io/docs/user/quick-start#installation)

```bash
~$ brew install kind
```

### 安裝 helm
K8s 套件管理工具，非 macOS 安裝方式請參閱[官方文件](https://helm.sh/docs/intro/install/)

```bash
~$ brew install helm
```

### 安裝 kubectl
K8s CLI 工具，用來與 K8s Cluster API Server 溝通，非 macOS 安裝方式請參閱[官方文件](https://kubernetes.io/docs/tasks/tools/#kubectl)

```bash
~$ brew install kubectl
```

### 建立 K8s Cluster

```bash
~$ unset DOCKER_DEFAULT_PLATFORM
~$ kind create cluster --config kind-config.yaml

# 執行完以上指令後，會看到類似底下的訊息
Creating cluster "genai-demo" ...
 ✓ Ensuring node image (kindest/node:v1.31.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-genai-demo"
You can now use your cluster with:

kubectl cluster-info --context kind-genai-demo

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

### 切換 K8s Cluster

```bash
~$ kind export kubeconfig --name genai-demo --kubeconfig $HOME/.kube/config
```

### 安裝 Prometheus Stack
加入 helm chart repository 後，就可以透過 helm 安裝 Prometheus Stack

```bash
~$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

"prometheus-community" has been added to your repositories

# 成功加入後，執行 helm repo update
~$ helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "k8sgpt" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈

# 安裝 Prometheus Stack
~$ helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
   --namespace monitoring \
   --create-namespace \
   --values prometheus-stack-values.yaml

# 執行完以上指令後，會看到類似底下的訊息
NAME: kube-prometheus-stack
LAST DEPLOYED: Tue Oct 22 18:45:02 2024
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=kube-prometheus-stack"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

### 確認 Prometheus Stack 是否正常運作

```bash
~$ kubectl get pod -n=monitoring

# 確認 Prometheus Stack 是否正常運作
NAME                                                        READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   0          98s
kube-prometheus-stack-grafana-b68d76fcb-rrgzh               3/3     Running   0          119s
kube-prometheus-stack-kube-state-metrics-6d57649657-8s4fm   1/1     Running   0          119s
kube-prometheus-stack-operator-7b4d8bdcc9-2xnr4             1/1     Running   0          119s
kube-prometheus-stack-prometheus-node-exporter-9m8lk        1/1     Running   0          119s
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0          98s
```

## 分析問題造成原因

### 環境設定
#### 安裝 k8sgpt
底下為 macOS 安裝指令，其他作業系統請參閱[官方文件](https://github.com/k8sgpt-ai/k8sgpt#cli-installation)

```bash
~$ brew install k8sgpt
```

#### 安裝 k8sgpt operator

```bash
~$ helm repo add k8sgpt https://charts.k8sgpt.ai/
~$ helm repo update
~$ helm install release k8sgpt/k8sgpt-operator \
   --namespace k8sgpt-operator-system \
   --create-namespace

# 執行完以上指令後，會看到類似底下的訊息
NAME: release
LAST DEPLOYED: Tue Oct 22 16:02:40 2024
NAMESPACE: k8sgpt-operator-system
STATUS: deployed
REVISION: 1
TEST SUITE: None

# 確認安裝狀態
~$ kubectl get pod -n k8sgpt-operator-system

NAME                                                          READY   STATUS              RESTARTS   AGE
release-k8sgpt-operator-controller-manager-7bfc5f9574-7qw7c   2/2     Running   0          64s
```

#### 設定 openai API Key
替換 `OPENAI_TOKEN` 為你的 OpenAI API Key

```bash
~$ export OPENAI_TOKEN=sk-xxxxxx

~$ kubectl create secret generic k8sgpt-sample-secret --from-literal=openai-api-key=$OPENAI_TOKEN -n k8sgpt-operator-system

~$ k8sgpt auth add --backend openai --model gpt-4o --password $OPENAI_TOKEN
```

#### 新增 k8sgpt 設定檔
```bash
~$ kubectl apply -f - << EOF
apiVersion: core.k8sgpt.ai/v1alpha1
kind: K8sGPT
metadata:
  name: k8sgpt-sample
  namespace: k8sgpt-operator-system
spec:
  ai:
    enabled: true
    model: gpt-4o
    backend: openai
    secret:
      name: k8sgpt-sample-secret
      key: openai-api-key
  noCache: false
  repository: ghcr.io/k8sgpt-ai/k8sgpt
  version: v0.3.41
EOF
```

#### 確認 k8sgpt 是否正常運作

```bash
~$ kubectl get results -n k8sgpt-operator-system -o json | jq .

# 安裝正確的話，會看到如下訊息：
{
    "apiVersion": "v1",
    "items": [],
    "kind": "List",
    "metadata": {
        "resourceVersion": ""
    }
}
```

### Demo

#### 部署有問題的 Nginx Deployment

```bash
~$ kubectl apply -f error-nginx.deployment.yaml

# 部署完後，可以看到 Pod 進入 CrashLoopBackOff 狀態
~$ kubectl get pod

NAME                               READY   STATUS         RESTARTS   AGE
nginx-deployment-ddd476688-flr5r   0/1     ErrImagePull   0          8s
```
#### 問題分析

```bash
~$ k8sgpt analyse

# k8sgpt 分析後，會看到如下訊息：
AI Provider: AI not used; --explain not set

0: Pod default/nginx-deployment-ddd476688-flr5r(Deployment/nginx-deployment)
- Error: Back-off pulling image "nginx:1.999.2"
```

#### 透過 GenAI 嘗試找尋解決方案

```bash
~$ k8sgpt analyse --explain

 100% |████████████████████████████████████████████████████████████████| (1/1, 47 it/min)
AI Provider: openai

0: Pod default/error-nginx-deployment-57cb6cd469-k4wj2(Deployment/error-nginx-deployment)
- Error: Back-off pulling image "nginx:1.999.2"
Error: Kubernetes is unable to pull the Docker image "nginx:1.999.2" because it likely doesn't exist.

Solution:
1. Verify the image tag is correct.
2. Check Docker Hub or your image registry for the correct version.
3. Update your Kubernetes configuration with a valid image tag.
```

## 操作指令自動生成

### 環境設定

#### 安裝 kubectl-ai
```bash
~$ brew tap sozercan/kubectl-ai https://github.com/sozercan/kubectl-ai
~$ brew install kubectl-ai
```

#### 設定 openai API Key
替換 `OPENAI_API_KEY` 為你的 OpenAI API Key

```bash
~$ export OPENAI_API_KEY=sk-xxxxxx
~$ export OPENAI_DEPLOYMENT_NAME=gpt-4o
```

### Demo

### 部署一個 Nginx Deployment

```bash
~$ kubectl ai "create an nginx deployment with 3 replicas"

⣟ Processing...
  ✨ Attempting to apply the following manifest:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
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
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80

Use the arrow keys to navigate: ↓ ↑ → ←
? (context: kind-genai-demo) Would you like to apply this? [Reprompt/Apply/Don't Apply]:
+   Reprompt
  ▸ Apply
    Don't Apply
```

### 確認 Nginx Pod 是否正常運作

```bash
~$ kubectl get pod

NAME                                      READY   STATUS             RESTARTS   AGE
error-nginx-deployment-57cb6cd469-2jd66   0/1     ImagePullBackOff   0          49s
nginx-deployment-54b9c68f67-7hknp         1/1     Running            0          10s
nginx-deployment-54b9c68f67-s5j4s         1/1     Running            0          10s
nginx-deployment-54b9c68f67-vqb6p         1/1     Running            0          10s
```

## 系統安全弱點建議

### 環境設定

#### 安裝 Trivy

```bash
~$ helm repo add aqua https://aquasecurity.github.io/helm-charts/
~$ helm repo update
~$ helm install trivy-operator aqua/trivy-operator \
   --namespace trivy-system \
   --create-namespace \
   --version v0.24.1 \
   --values trivy-values.yaml
```

#### 開啟 k8sgpt Trivy 整合

```bash
~$ k8sgpt integration activate trivy --no-install --namespace trivy-system
```

### Demo

#### 查看系統安全弱點

```bash
k8sgpt analyze --filter VulnerabilityReport
```

#### 查看系統安全弱點建議
```bash
k8sgpt analyse --filter=VulnerabilityReport --explain
```

## 快速解決線上問題

### 環境設定

#### 安裝 robusta
根據[官方網頁](https://docs.robusta.dev/master/setup-robusta/installation/extend-prometheus-installation.html)指示申請帳號，並且整合 Slack 後，會獲得 `generated_values.yaml` 檔案，然後透過 helm 安裝 robusta

```bash
~$ helm repo add robusta https://robusta-charts.storage.googleapis.com
~$ helm repo update
~$ helm install robusta robusta/robusta \
   --namespace robusta-system \
   --create-namespace \
   --set clusterName=genai-demo \
   --set isSmallCluster=true \
   --values ./generated_values.yaml
```

#### 新增 alertmanagerconfig CRD

```bash
kubectl apply -f alertmanagerconfig.yaml -n monitoring
```
### Demo

#### 部署錯誤的 K8s Deployment

```bash
kubectl apply -f https://gist.githubusercontent.com/robusta-lab/283609047306dc1f05cf59806ade30b6/raw -n default
```

根據 [官方文件](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/alerting.md#using-alertmanagerconfig-resources) 修改 `spec.alertmanagerConfigSelector`

```bash
~$ kubectl edit alertmanager kube-prometheus-stack-alertmanager -n monitoring

...
   alertmanagerConfigSelector:
     matchLabels:
       alertmanagerConfig: example
...
```