# projet-koala
HA + Automatisation

## Test verification

### Test rôle: all
```bash
ansible-playbook -i inventory/hosts.ini -k site.yml
```

**VALIDER SUR INFRA**

### Test role: Loki + Promtail + Grafana

**VALIDER SUR INFRA**

### Test role: NFS + Web

**VALIDER SUR INFRA** : Installation uniquement

### Test role: BDD + Galera