# Ansible MicroK8s Cluster

Modul Ansible pentru instalarea și configurarea unui cluster MicroK8s pe 3 noduri.

## 📋 Structura proiectului

```
ansible-microk8s-cluster/
├── ansible.cfg              # Configurație Ansible
├── inventory/
│   └── hosts.ini            # Definirea nodurilor (IP-uri, credentials)
├── group_vars/
│   └── all.yml              # Variabile globale (addons, tool-uri)
├── roles/
│   ├── common/              # Pregătire sistem (sysctl, module kernel)
│   ├── microk8s/            # Instalare MicroK8s
│   ├── cluster/             # Formare cluster (join nodes)
│   └── extras/              # Tool-uri adiționale (k9s, kubectx, etc.)
├── playbooks/
│   ├── site.yml             # Instalare completă
│   ├── prepare.yml          # Doar pregătire sistem
│   ├── reset.yml            # Dezinstalare completă
│   ├── add-node.yml         # Adăugare nod nou
│   └── diagnose.yml         # Diagnostic cluster
└── README.md
```

## 🖥️ Noduri configurate

| Rol       | Hostname   | IP Address     |
|-----------|------------|----------------|
| Master    | master-1   | 192.168.0.61   |
| Worker    | worker-1   | 192.168.0.154  |
| Worker    | worker-2   | 192.168.0.229  |

## ⚙️ Cerințe

- **Ansible** >= 2.9 instalat pe mașina locală
- **SSH access** la toate nodurile cu cheia `~/.ssh/openstack-cluster-key.pem`
- **Ubuntu/Debian** pe nodurile țintă (testat pe Ubuntu 22.04)
- **Privilegii sudo** fără parolă pe noduri

## 🚀 Instalare Ansible

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y ansible

# sau via pip
pip install ansible
```

## 📦 Utilizare

### 1. Verificare conectivitate

```bash
cd ansible-microk8s-cluster
ansible all -m ping
```

### 2. Instalare completă cluster

```bash
ansible-playbook playbooks/site.yml
```

### 3. Doar pregătire sistem (fără MicroK8s)

```bash
ansible-playbook playbooks/prepare.yml
```

### 4. Diagnostic cluster

```bash
ansible-playbook playbooks/diagnose.yml
```

### 5. Resetare completă (ATENȚIE!)

```bash
ansible-playbook playbooks/reset.yml
```

## 🔧 Addon-uri instalate

Următoarele addon-uri sunt activate automat:

- **dns** - CoreDNS pentru service discovery
- **dashboard** - Kubernetes Dashboard
- **ingress** - NGINX Ingress Controller
- **storage** - Local storage provisioner
- **metrics-server** - Metrici pentru HPA
- **rbac** - Role-Based Access Control
- **helm3** - Helm package manager
- **registry** - Container registry local
- **hostpath-storage** - PersistentVolume storage

## 🛠️ Tool-uri adiționale instalate

- **k9s** - Terminal UI pentru Kubernetes
- **kubectx** - Schimbare rapidă între contexte
- **kubens** - Schimbare rapidă între namespace-uri
- **stern** - Multi-pod log tailing

## 📝 Alias-uri disponibile

După instalare, sunt disponibile următoarele alias-uri:

```bash
k       → microk8s kubectl
kgp     → kubectl get pods
kgs     → kubectl get svc
kgn     → kubectl get nodes
kga     → kubectl get all
kgaa    → kubectl get all --all-namespaces
kdp     → kubectl describe pod
kds     → kubectl describe svc
kdn     → kubectl describe node
kl      → kubectl logs
klf     → kubectl logs -f
kex     → kubectl exec -it
kaf     → kubectl apply -f
kdf     → kubectl delete -f
kctx    → kubectx
kns     → kubens
```

## 🔐 Acces la Dashboard

```bash
# Obține token pentru dashboard
ssh -i ~/.ssh/openstack-cluster-key.pem ubuntu@192.168.0.61 \
  "microk8s kubectl -n kube-system describe secret \$(microk8s kubectl -n kube-system get secret | grep default-token | awk '{print \$1}')"

# Port forward pentru dashboard
ssh -i ~/.ssh/openstack-cluster-key.pem -L 10443:127.0.0.1:10443 ubuntu@192.168.0.61 \
  "microk8s kubectl port-forward -n kube-system svc/kubernetes-dashboard 10443:443"
```

Accesează: https://localhost:10443

## 📊 Verificare cluster

```bash
# Pe master
ssh -i ~/.ssh/openstack-cluster-key.pem ubuntu@192.168.0.61

# Verifică noduri
microk8s kubectl get nodes -o wide

# Verifică poduri sistem
microk8s kubectl get pods -A

# Status cluster
microk8s status
```

## 🔄 Adăugare nod nou

1. Adaugă nodul în `inventory/hosts.ini`:
```ini
[new_node]
worker-3 ansible_host=192.168.0.XXX
```

2. Rulează playbook-ul:
```bash
ansible-playbook playbooks/add-node.yml
```

## ⚠️ Troubleshooting

### Verificare conectivitate SSH
```bash
ssh -i ~/.ssh/openstack-cluster-key.pem ubuntu@192.168.0.61 "hostname"
```

### Verificare status pe un nod specific
```bash
ansible master-1 -m command -a "microk8s status"
```

### Log-uri MicroK8s
```bash
ssh -i ~/.ssh/openstack-cluster-key.pem ubuntu@192.168.0.61
journalctl -u snap.microk8s.daemon-kubelite -f
```

## 📚 Documente utile

- [MicroK8s Documentation](https://microk8s.io/docs)
- [Ansible Documentation](https://docs.ansible.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

## 📄 Licență

MIT License
