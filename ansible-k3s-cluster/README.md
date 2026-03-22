# Ansible K3s Cluster Deployment

Acest proiect Ansible automatizează instalarea și configurarea unui cluster Kubernetes folosind **K3s** (distribuția lightweight de la Rancher).

## Structura Proiectului

```
ansible-k3s-cluster/
├── ansible.cfg              # Configurare Ansible
├── inventory/
│   ├── hosts.ini            # Inventar noduri (EDITEAZĂ AICI IP-urile tale)
│   └── group_vars/
│       └── all.yml          # Variabile globale
├── playbooks/
│   ├── site.yml             # Playbook principal (instalare completă)
│   ├── reset.yml            # Dezinstalare simplă cluster
│   ├── nuke.yml             # ⚠️  Ștergere TOTALĂ și ireversibilă K3s
│   ├── calico.yml           # Instalare Calico CNI pe cluster
│   └── upgrade.yml          # Upgrade K3s
└── roles/
    ├── common/              # Pregătire sistem (swap, pachete)
    ├── k3s-server/          # Instalare nod master (server)
    ├── k3s-agent/           # Instalare noduri worker (agent)
    ├── k3s-uninstall/       # Dezinstalare completă (folosit de nuke.yml)
    ├── calico/              # Instalare Calico CNI (NetworkPolicy)
    └── extras/              # Tool-uri adiționale (helm, kubectl alias)
```

## Cerințe Prealabile

1. **Control Node (PC-ul tău):**
   - Ansible instalat (`pip install ansible`)
   - SSH key generat (`ssh-keygen`)

2. **Target Nodes (VM-urile/serverele):**
   - Ubuntu 22.04+ sau Debian 11+
   - SSH access cu utilizator care are sudo
   - Minim 2GB RAM per nod

## Configurare

### 1. Editează Inventarul
Modifică `inventory/hosts.ini` cu IP-urile nodurilor tale:

```ini
[k3s_masters]
master-node ansible_host=192.168.0.229

[k3s_workers]
worker-node-1 ansible_host=192.168.0.61
worker-node-2 ansible_host=192.168.0.154

[k3s_cluster:children]
k3s_masters
k3s_workers
```

### 2. Configurează Variabilele (Opțional)
Editează `inventory/group_vars/all.yml` pentru a personaliza instalarea.

### 3. Testează Conectivitatea
```bash
ansible -i inventory/hosts.ini all -m ping
```

## Utilizare

### Instalare Cluster Complet
```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml
```

### Doar Instalare Server (Master)
```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --tags "server"
```

### Doar Join Workers
```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --tags "agent"
```

### Dezinstalare Simplă
```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset.yml
```

### Instalare Calico CNI (Network Policy)

> Calico înlocuiește politicile de rețea built-in ale K3s. Rulează DOAR pe master — Calico DaemonSet se va extinde automat pe toți workerii.

```bash
ansible-playbook -i inventory/hosts.ini playbooks/calico.yml
```

Ce face:
- Configurează K3s pentru CNI extern:
   - `flannel-backend: none`
   - `disable-network-policy: true`
- Restartează K3s (~30-60s downtime)
- Verifică explicit că nodurile sunt `NotReady` (stare normală fără CNI)
- Instalează **Tigera Operator** (creierul Calico)
- Aplică **custom-resources** cu CIDR-ul corect (`10.42.0.0/16`)
- Așteaptă să fie toate pod-urile `calico-system` în starea `Running`

Pentru altă versiune Calico:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/calico.yml -e "calico_version=v3.31.1"
```

Verificare manuală după instalare:
```bash
ssh ubuntu@10.102.13.205 'sudo k3s kubectl get pods -n calico-system'
```

### ⚠️  Ștergere TOTALĂ (NUKE) — elimină K3s complet de pe toate nodurile

> **ATENȚIE:** Operațiunea este **ireversibilă**! Șterge tot: binare, date, rețea, iptables, helm.

```bash
# Rulare cu confirmare obligatorie:
ansible-playbook -i inventory/hosts.ini playbooks/nuke.yml -e "confirm_nuke=yes"
```

Ce face comanda NUKE:
- 🛑 Oprește serviciile `k3s` și `k3s-agent`
- 🗑️ Rulează script-urile oficiale de dezinstalare (`k3s-uninstall.sh`, `k3s-agent-uninstall.sh`)
- 🔪 Omoară procese reziduale (containerd, flannel)
- 📁 Șterge toate datele (`/var/lib/rancher`, `/etc/rancher`, `/var/lib/kubelet`, `/var/lib/cni`)
- 🔌 Elimină interfețele de rețea CNI/flannel (`cni0`, `flannel.1`, `vxlan`)
- 🔒 Resetează regulile iptables la starea inițială
- 📦 Unmountează volume reziduale Kubernetes
- 🪖 Șterge Helm (dacă a fost instalat)
- 🧹 Curăță `~/.kube`, `.bashrc` și fișierele unit systemd

După NUKE, poți reinstala clusterul cu:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml
```

## După Instalare

Kubeconfig-ul va fi copiat automat în `~/.kube/config` pe nodul master.

Pentru a accesa clusterul de pe PC-ul tău:
```bash
scp ubuntu@<master-ip>:~/.kube/config ~/.kube/config
kubectl get nodes
```

## Variabile Importante

| Variabilă | Default | Descriere |
|-----------|---------|-----------|
| `k3s_version` | `v1.35.2+k3s1` | Versiunea K3s de instalat |
| `k3s_server_args` | `--disable traefik` | Argumente suplimentare pentru server |
| `install_helm` | `true` | Instalează Helm 3 |
| `install_longhorn` | `false` | Instalează Longhorn pentru storage |
