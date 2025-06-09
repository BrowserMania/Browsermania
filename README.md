# 🌐 BROWSERMANIA

![Logo](Browsermania.png)

*Solution de navigation web isolée*

![dernier-commit](https://img.shields.io/github/last-commit/sony-level/Browsermania?style=flat&logo=git&logoColor=white&color=0080ff)
![langage-principal-repo](https://img.shields.io/github/languages/top/sony-level/Browsermania?style=flat&color=0080ff)
![nombre-langages-repo](https://img.shields.io/github/languages/count/sony-level/Browsermania?style=flat&color=0080ff)

---

## 🧭 Browsermania : solution de navigation web isolée

**Browsermania** est une solution de navigation web **sécurisée, isolée et orchestrée**.

Chaque session navigateur est exécutée dans un **conteneur dédié et cloisonné**, contrôlé via Kubernetes, avec accès distant par **WebRTC** .

### ✅ Principaux atouts

- 🔐 Navigation web **conteneurisée, isolée **
- 🌐 Accès distant via **WebRTC**
- 🧱 Cloisonnement réseau avec **Cilium**
- 🚀 Déploiement complet d’une stack backend/frontend

---

## 📚 Table des matières

- [Présentation](#-présentation)
- [Démarrage rapide](#-démarrage-rapide)
    - [Prérequis](#-prérequis)
    - [Installation](#-installation)
    - [Configuration du cluster](#-configuration-du-cluster)
- [Tests](#-tests)
- [Sécurité et bonnes pratiques](#-sécurité-et-bonnes-pratiques)
- [Licence](#-licence)

---

## ✨ Présentation

Browsermania est un projets qui permet de déployer :

- Un Interface web (frontend)  configuré 
- Un Backend (API) pour la gestion des sessions de navigation
- Un Serveur WebRTC pour accès distant de navigateur conteneurisé. 
- Un environnement Kubernetes complet (MetalLB, Cilium, stockage...)

---
## 🚀 Démarrage rapide

### 📋 Prérequis

- OS : Ubuntu/Debian
- kata-container installer
- Accès root ou `sudo`
- Internet
- `minikube` ou `kubeadm`
- Docker ou Containerd

### ⚙️ Installation

```bash
git clone https://github.com/sony-level/Browsermania
cd Browsermania
````
### 🧰 Configuration du cluster

Script de configuration d'un cluster Kubernetes avec Containerd, Cilium et Cilium CLI

Ce script shell configure un nœud (maître ou worker) pour un cluster Kubernetes basé sur Containerd avec le CNI Cilium.

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
```
### Post-installation

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

