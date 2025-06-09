#           # 🌐 Browsermania

> Solution de navigation web isolée, sécurisée et orchestrée via Kubernetes.

---

## 🧭 Présentation

**Browsermania** est une solution de navigation web sécurisée, isolée et orchestrée. Chaque session de navigation est exécutée dans un conteneur dédié, contrôlé via Kubernetes, et accessible à distance via WebRTC. Le projet intègre **Cilium** pour le cloisonnement réseau et **MetalLB** pour le load balancing.

### ✅ Principaux atouts

- 🔐 Navigation web conteneurisée et isolée
- 🌐 Accès distant via WebRTC
- 🧱 Cloisonnement réseau avec Cilium
- 🚀 Déploiement complet d’une stack backend/frontend

---

## 📚 Table des matières

- [Présentation](#-présentation)
- [Démarrage rapide](#-démarrage-rapide)
    - [Prérequis](#prérequis)
    - [Installation](#installation)
    - [Configuration du cluster](#configuration-du-cluster)
        - [Option 1 : Minikube](#option-1--minikube)
        - [Option 2 : Kubeadm](#option-2--kubeadm)
- [Tests](#tests)
- [Dépannage](#dépannage)

---

## 🚀 Démarrage rapide

### Prérequis

- OS : Ubuntu ou Debian
- Accès root ou sudo
- Outils :
    - Minikube ou Kubeadm
    - Docker ou Containerd
    - `kubectl`
    - Paquets : `curl`, `apt-transport-https`, `ca-certificates`, `software-properties-common`
- Connexion Internet active

---

### Installation

```bash
git clone https://github.com/sony-level/Browsermania
cd Browsermania

````
## ⚙️ Configuration du cluster

### Option 1 : Minikube

0. **configuration via le script d'installation**  
   Vous pouvez utiliser le script bash fourni pour configurer un cluster Kubernetes avec Minikube,  et MetalLB.  
   👉 [https://github.com/sony-level/Browsermania/blob/main/buil_with_minikube](https://github.com/sony-level/Browsermania/blob/main/build_with_minikube)

1. **Installation de Minikube**  
   Consultez la documentation officielle :  
   👉 [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/)

2. **Démarrage avec Containerd**
```bash
minikube start --driver=docker --container-runtime=containerd --network-plugin=cni \
--extra-config=kubelet.container-runtime-endpoint=unix:///var/run/containerd/containerd.sock
```
3. **Installation de Cilium CLI**
   ```bash
   curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
   sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
   sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
   rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
   ```
`4. **Installation de Cilium dans Minikube**
   ```bash
   cilium install
   ```
5. **Installation de MetalLB**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
   ```
6. **Configuration de MetalLB**
   ```bash
   kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
   ```
7. **Déploiement de Browsermania**
   ```bash
   kubectl apply -f ./deploy_browsermania.yaml
   ```
8. **Vérification du déploiement**
   ```bash
   kubectl get pods
   kubectl get svc
   ``` 
### Option 2 : Kubeadm
1. **Installation de Kubeadm, Kubelet et Kubectl**
   Suivez les instructions officielles :  
   👉 [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

ou 
2. **Utilisation du script d'installation**
   Vous pouvez utiliser le script bash fourni pour configurer un cluster Kubernetes avec containerd, Cilium et Cilium CLI.  
   👉 [https://github.com/sony-level/Browsermania/blob/main/buil_with_kubedm](https://github.com/sony-level/Browsermania/blob/main/buil_with_kubedm)

#### 📜 Script Bash (extrait)
```bash
#!/bin/bash
# Script pour configurer un cluster Kubernetes avec containerd, Cilium et Cilium CLI

# Exécuter sur tous les nœuds
echo "Mise à jour du système et installation des dépendances..."
sudo apt update && sudo apt upgrade -y
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common

echo "Activation de l'IP forwarding..."
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

echo "Installation de containerd..."
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

echo "Ajout du dépôt Kubernetes..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update

echo "Installation de kubeadm, kubelet et kubectl..."
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "Configuration de kubelet pour utiliser containerd..."
cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS=--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock
EOF

sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl restart kubelet

echo "Installation de Cilium CLI..."
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Initialisation uniquement sur le nœud maître
if [ "$1" == "master" ]; then
  echo "Initialisation du nœud maître Kubernetes..."
  sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
fi
``

Une fois ce script executé et le cluster initialisé, lancer :
```bash
cilium install
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml (ou une version plus récente)
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl apply -f ./deploy_browsermania.yaml
```
### 🧪 Tests
Pour tester le déploiement, vous pouvez utiliser les commandes suivantes :

```bash
kubectl get pods 
kubectl get svc
```

> **ℹ️ Important :** Les services de type `LoadBalancer` doivent être exposés via MetalLB et accessibles sur l'IP allouée.



