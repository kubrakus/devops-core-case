# 1️⃣ Kubernetes Cluster Kurulumu
Bu bölümde **1 master + 1 worker** node’dan oluşan Kubernetes cluster kurulum adımları yer almaktadır.
Ubuntu 22.04 sanal sunucular UTM aracılığıyla kurulmuştur.   

- master: 2 CPU, 4 GB RAM

- worker: 2 CPU, 4 GB RAM

Network yapısı için **Cilium CNI** kullanılmıştır.

#### Versiyon Seçimi

Kurulumda **Kubernetes v1.32.8** sürümü tercih edilmiştir. Bunun nedeni, en güncel sürümü kullanmak yerine bilinen hataları giderilmiş, **stabil ve sorunsuz çalışan** bir versiyon seçme isteğidir.  
**Cilium v1.17.4** ise bu Kubernetes sürümü ile resmi uyumluluk tablosuna göre tam destek veren stabil CNI versiyonu olduğu için tercih edilmiştir.

---

### 1. Ön Hazırlık (Tüm Nodelarda)
Tüm node’larda aşağıdaki adımlar uygulanır:

```bash
# /etc/hosts dosyasına master ve worker IP/hostname eklenir.
echo "192.168.64.2  k8s-master" | sudo tee -a /etc/hosts
echo "192.168.64.3  k8s-worker" | sudo tee -a /etc/hosts

# Paketler güncellenir.
sudo apt update && sudo apt -y upgrade

# Swap devre dışı bırakılır.
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Eğer sed ile kapatılamazsa vi ile manuel aşağıdaki gibi kapatılabilir.
sudo vi /etc/fstab 
```

### 2. Containerd Kurulumu ve Ayarları (Tüm Node’lar)

```bash
# Containerd kurulumu
sudo apt install -y containerd

# Default config oluşturma
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Pause image versiyonunun güncellenmesi (ARM64 uyumu için)
sudo sed -i 's#sandbox_image = "registry.k8s.io/pause:3.8"#sandbox_image = "registry.k8s.io/pause:3.10"#' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 3. Kubernetes Bileşenlerinin Kurulumu (kubeadm, kubelet, kubectl) (Tüm Node'lar)

```bash

# Gerekli paketler
sudo apt install -y apt-transport-https ca-certificates curl

# Kubernetes GPG key ekleme
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Kubernetes repo ekleme
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Paket listesi güncelleme
sudo apt update

# Kubernetes paketlerini kurma (ARM64 uyumlu)
sudo apt install -y kubelet kubeadm kubectl

# Paketlerin versiyonunu sabitleme
sudo apt-mark hold kubelet kubeadm kubectl
```

### 4. Master Node Kurulumu

```bash
# Master node init
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Kullanıcıya kubeconfig ayarlama
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Not: Root kullanıcı kullanılıyorsa;
export KUBECONFIG=/etc/kubernetes/admin.conf

```
