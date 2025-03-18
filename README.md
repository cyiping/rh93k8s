# New install in Nokia lab

## RHEL 9.3 部署 Kubernetes 1.32（含 Flannel、Metrics Server、Dashboard）


### 系統需求

✅ RHEL 9.3（Master 與 Worker 節點）\
✅ Kubernetes 1.32\
✅ `containerd` 作為容器運行時\
✅ `Flannel`（網路插件）\
✅ `Metrics Server`（監控）\
✅ `Kubernetes Dashboard`（管理界面）

---

## **步驟 1：關閉 Swap 及 SELinux**

```bash
# 關閉 Swap
sudo swapoff -a

# 永久禁用 Swap
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 關閉 SELinux
sudo setenforce 0

# 永久禁用 SELinux
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

---

## **步驟 2：安裝 containerd**

```bash
# 安裝 containerd
sudo yum install -y containerd.io

# 建立預設配置
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# 啟用 containerd
sudo systemctl enable --now containerd
```

---

## **步驟 3：安裝 `kubeadm`, `kubelet`, `kubectl`**

```bash
# 添加 Kubernetes 倉庫
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF

# 安裝 Kubernetes 套件
sudo yum install -y kubelet kubeadm kubectl

# 啟用 kubelet
sudo systemctl enable --now kubelet
```

---

## **步驟 4：初始化 Kubernetes 集群**

```bash
# 設定 Master 節點 IP（請換成實際 IP）
MASTER_IP="192.168.1.100"

# 初始化 Kubernetes
sudo kubeadm init \
  --apiserver-advertise-address=$MASTER_IP \
  --pod-network-cidr=100.64.0.0/10 \
  --service-cluster-ip-range=10.96.0.0/22
```

初始化完成後，執行以下指令讓 `kubectl` 可用：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## **步驟 5：安裝 Flannel（網路插件）**

```bash
# 下載 Flannel YAML 配置檔
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**檢查 Flannel 是否成功部署**

```bash
kubectl get pods -n kube-flannel
```

---

## **步驟 6：加入 Worker 節點**

在 **Master 節點** 執行：

```bash
kubeadm token create --print-join-command
```

這將輸出類似：

```bash
kubeadm join 192.168.1.100:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

然後，在 **Worker 節點** 執行：

```bash
sudo kubeadm join 192.168.1.100:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## **步驟 7：安裝 Metrics Server（監控）**

```bash
# 安裝 Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**驗證 Metrics Server 是否正常運作**

```bash
kubectl get deployment metrics-server -n kube-system
```

---

## **步驟 8：安裝 Kubernetes Dashboard（管理界面）**

```bash
# 安裝 Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### **建立管理員帳戶**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

**取得登入 Token**

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

**啟動 Dashboard**

```bash
kubectl proxy
```

然後在瀏覽器中開啟：

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

輸入剛剛取得的 Token，即可登入 **Kubernetes Dashboard**。

---

## **結論**

✅ **符合 Kubernetes 1.32，適用於 RHEL 9.3**\
✅ **完整安裝 `Flannel`（網路）、`Metrics Server`（監控）、`Dashboard`（管理）**\
✅ **已設定 `service-cluster-ip-range=10.96.0.0/22` 和 `cluster-cidr=100.64.0.0/10`**\
✅ **提供完整的 Worker 加入方式與測試指令**\
✅ **新增關閉 Swap 及 SELinux 步驟，確保 Kubernetes 正常運作**

這樣你就可以 **快速部署一個可管理的 Kubernetes 叢集！** 🚀




