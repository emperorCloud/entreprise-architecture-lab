# Proxmox VE – Architecture d'entreprise

## 📌 Contexte d'utilisation

> *Quand Proxmox VE est-il le meilleur choix ?*

| Critère | Détail |
|---------|--------|
| **Type d'organisation** | PME, ETI, établissement public, éducation |
| **Taille typique** | 3 à 50 serveurs physiques |
| **Budget** | Faible à modéré |
| **Complexité** | Modérée |
| **Compétences requises** | Linux avancé, notions de stockage distribué |

### Cas d'usage idéaux

- Virtualisation on-premise sans licence
- Infrastructure hyperconvergée (compute + stockage sur les mêmes nœuds)
- Labs de développement, POC, pré-production
- Remplacement d'un VMware vSphere trop coûteux
- Data centers de taille modeste (Tier II-III)

### Quand NE PAS utiliser Proxmox

- Grande entreprise avec 100+ hyperviseurs nécessitant vCenter, DRS, NSX
- Environnement 100% Windows sans compétences Linux
- Contrainte de support éditeur 24/7 avec SLA (existe via partenaires mais coûteux)
- Intégration obligatoire avec des outils VMware (Veeam, Zerto, etc.)

---

## 🧠 Architecture de référence

### Design global – Cluster Hyperconvergé 3 nœuds

Ce design représente l'architecture la plus courante et recommandée en production.

**Caractéristiques :**
- 3 nœuds Proxmox VE en cluster
- Stockage Ceph distribué sur les mêmes nœuds (hyperconvergence)
- Double attache réseau : 1 lien management/Corosync, 1 lien Ceph/storage
- Haute disponibilité activée pour les VMs critiques
- Sauvegarde externalisée sur PBS (Proxmox Backup Server)

![Architecture Proxmox Ceph 3 nœuds](images/proxmox-ceph-3nodes.png)

### Flux réseau

| Réseau | Plage | Rôle |
|--------|-------|------|
| Management | 10.0.0.0/24 | Interface web, SSH, Corosync (anneau 1) |
| Ceph Public | 10.0.1.0/24 | Accès client au stockage Ceph |
| Ceph Cluster | 10.0.2.0/24 | Réplication inter-nœuds Ceph (back-end) |
| VMs / VLANs | Selon besoins | Réseau des machines virtuelles |

### Composants

| Composant | Rôle | Justification |
|-----------|------|---------------|
| **Proxmox VE 8.x** | Hyperviseur KVM + LXC | Open source, mature, interface web complète |
| **Corosync** | Communication du cluster, quorum | Tolérance de panne, HA |
| **Ceph** | Stockage distribué répliqué | Pas de SAN externe, pas de SPOF |
| **Proxmox Backup Server** | Sauvegarde incrémentielle, déduplication | Sauvegardes rapides, restauration fine |
| **ZFS** (optionnel) | Stockage local pour VMs non-critiques | Performances, snapshots, compression |

---

## ⚖️ Analyse décisionnelle

### Pourquoi Proxmox VE ?

1. **Licensing simple et transparent**
   - Pas de surprise Broadcom, pas de calcul au core, pas de socket.
   - Support communautaire gratuit, support éditeur en option.

2. **KVM natif, pas de surcouche**
   - Performances proches du bare-metal (à 3-5% près).
   - Pas de couche de compatibilité comme Hyper-V ou VMware.

3. **Hyperconvergence intégrée**
   - Ceph et ZFS sont intégrés dans l'interface.
   - Pas de SAN, pas de baie externe = réduction du coût et des SPOF.

4. **Double nature VMs + conteneurs**
   - LXC pour les services légers, KVM pour les OS complets.
   - Un seul outil pour deux besoins.

5. **Sauvegardes professionnelles intégrées**
   - PBS offre déduplication, chiffrement, vérification, restauration granulaire.
   - Pas de licence supplémentaire.

### Pourquoi pas les alternatives ?

| Alternative | Pourquoi pas ? |
|-------------|----------------|
| **VMware vSphere** | Coût prohibitif depuis rachat Broadcom. Roadmap incertaine. Licensing complexe. |
| **Hyper-V** | Dépendance Windows. Moins performant en environnement Linux. Stockage moins flexible. |
| **XCP-ng** | Excellente alternative, mais Ceph nécessite XOSTOR (supplément). Communauté plus petite. |
| **Nutanix CE** | Version communauté limitée à 4 nœuds, pas de compression, pas de déduplication. Verrouillage éditeur. |
| **OpenStack** | Complexité massive. Justifié uniquement pour du cloud privé multi-tenant à grande échelle. |

---

## 📊 Matrice de décision

| Critère | Poids | Proxmox | VMware vSphere | XCP-ng | Hyper-V |
|---------|-------|---------|----------------|--------|---------|
| Coût licence | 30% | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Performance | 20% | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Facilité d'administration | 15% | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| Écosystème / Support | 15% | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| Stockage intégré | 20% | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Score pondéré** | **100%** | **4.35** | **2.85** | **3.60** | **2.90** |

> Ce tableau reflète un contexte **PME/ETI, budget maîtrisé, compétences Linux internes**. Pour un grand groupe avec 500+ VMs et besoin de support 24/7, les coefficients changent.

---

## 🔗 Interconnexions

Cette architecture s'intègre avec :

- **[Ceph](../../03-Storage/Software-Defined/Ceph/README.md)** – Stockage distribué
- **[ZFS](../../03-Storage/Software-Defined/ZFS/README.md)** – Stockage local
- **[pfSense](../../05-Security/pfSense/README.md)** – Pare-feu et routage
- **[HAProxy + Keepalived](../../08-HighAvailability/HAProxy/README.md)** – Load balancing externe
- **[Grafana + Prometheus](../../10-Monitoring/Grafana/README.md)** – Supervision
- **[Proxmox Backup Server](../../09-DisasterRecovery/Backup/README.md)** – Sauvegardes
- **[Ansible](../../12-Automation/Ansible/README.md)** – Déploiement automatisé

---

## 📈 Bonnes pratiques

### ✅ À faire

- [ ] **Minimum 3 nœuds** pour le cluster Ceph (taille de réplication 3/2)
- [ ] **Double attache réseau** physique séparée pour Corosync + Ceph
- [ ] **Disques d'OS sur SSD dédié**, pas sur le pool Ceph
- [ ] **Proxmox Backup Server sur un hôte physique distinct**
- [ ] **Monitoring Ceph activé** : alertes sur le remplissage, la santé, la latence
- [ ] **Mises à jour régulières** : Proxmox a un cycle rapide (2-3 releases/an sans downtime)
- [ ] **SSD NVMe** pour les WAL/DB Ceph si les performances sont critiques

### ❌ À éviter

- [ ] Cluster à 2 nœuds : pas de quorum, split-brain inévitable
- [ ] Ceph sur 1 Gbps seulement : la réplication sera lente, les performances dégradées
- [ ] Mélanger disques durs et SSD dans le même pool Ceph sans règles CRUSH
- [ ] Sauvegarder les VMs sur le même cluster que la production
- [ ] Ignorer les mises à jour de sécurité du noyau KVM
- [ ] Utiliser le root pour tout : créer des rôles et permissions

---

## 🚀 Déploiement résumé

```bash
# Installation de base Proxmox VE (ISO bootable)
# Puis configuration post-installation :

# 1. Désactiver le repo enterprise si pas de licence
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# 2. Mettre à jour
apt update && apt dist-upgrade -y

# 3. Créer le cluster
pvecm create NOM-DU-CLUSTER

# 4. Sur les autres nœuds
pvecm add IP-DU-PREMIER-NOEUD

# 5. Initialiser Ceph via l'interface web
# Datacenter > Ceph > Installation
