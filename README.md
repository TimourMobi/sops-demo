# SOPS Demo mit Flux GitOps

Dieses Repository demonstriert die Verwendung von SOPS (Secrets OPerationS) mit Flux GitOps für sichere Kubernetes Secret-Verwaltung.

## Voraussetzungen

- `age` Tool installiert (`brew install age`)
- `sops` Tool installiert (`brew install sops`)
- `k3d` für lokale Kubernetes Cluster (`brew install k3d`)
- `kubectl` für Kubernetes CLI (`brew install kubectl`)
- `flux` CLI für Flux GitOps (`brew install fluxcd/tap/flux`)

## Step-by-Step Setup

### 1. Age Key erstellen

```bash
# Age Key-Pair generieren
age-keygen -o age-key.txt

# Public Key anzeigen (für .sops.yaml)
cat age-key.txt | grep "public key"
```

### 2. SOPS Konfiguration

Die `.sops.yaml` wurde bereits konfiguriert mit dem Age Public Key:

```yaml
creation_rules:
  - path_regex: .*\.yaml$
    encrypted_regex: '^(data|stringData)$'
    age: age1rrfpte79wdj8tv03c8k7t7sf66vpygz4ydfndqne6vvsscmlhfjswvez9u
```

### 3. Secret verschlüsseln

```bash
# Unverschlüsseltes Secret erstellen (Beispiel)
cat << EOF > grafana-secret-plain.yaml
apiVersion: v1
kind: Secret
metadata:
    name: grafana-admin
    namespace: monitoring
type: Opaque
data:
    admin-user: YWRtaW4=  # admin (base64)
    admin-password: c2VjcmV0MTIz  # secret123 (base64)
EOF

# Mit SOPS verschlüsseln
export SOPS_AGE_KEY_FILE=age-key.txt
sops --encrypt --in-place grafana-secret-plain.yaml

# Verschlüsseltes Secret nach monitoring/ verschieben
mv grafana-secret-plain.yaml monitoring/grafana-secret.yaml
```

### 4. k3d Cluster erstellen

```bash
# k3d Cluster mit Ingress Controller erstellen
k3d cluster create sops-demo \
  --agents 2 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer"

# Nginx Ingress Controller installieren
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### 5. Flux Bootstrap

```bash
# Flux in den Cluster bootstrappen
flux bootstrap git \
  --url=https://github.com/TimourMobi/sops-demo \
  --branch=main \
  --path=flux-system

# SOPS Age Key als Secret hinzufügen
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age-key.txt
```

### 6. Flux für SOPS konfigurieren

```bash
# Flux Kustomization für SOPS erweitern
kubectl patch kustomization flux-system \
  --namespace flux-system \
  --type merge \
  --patch '{"spec":{"decryption":{"provider":"sops","secretRef":{"name":"sops-age"}}}}'
```

### 7. Deployment testen

```bash
# Pods prüfen
kubectl get pods -n monitoring

# Grafana Service prüfen
kubectl get svc -n monitoring

# Grafana über Ingress testen
echo "127.0.0.1 grafana.local" | sudo tee -a /etc/hosts
curl -H "Host: grafana.local" http://localhost:8080
```

### 8. Grafana Login

- **URL**: http://grafana.local:8080 (über k3d Port-Forwarding)
- **Username**: `admin`
- **Password**: `secret123`

## Repository Struktur

```
.
├── .sops.yaml                    # SOPS Konfiguration
├── README.md                     # Diese Datei
├── flux-system/
│   └── kustomization.yaml        # Flux Root Kustomization
└── monitoring/
    ├── namespace.yaml            # Monitoring Namespace
    ├── helm-repository.yaml      # Grafana Helm Repository
    ├── grafana-secret.yaml       # SOPS-verschlüsseltes Secret
    ├── grafana-helmrelease.yaml  # Grafana Helm Release
    ├── grafana-ingress.yaml      # Grafana Ingress
    └── kustomization.yaml        # Monitoring Kustomization
```

## Wichtige Befehle

```bash
# Secret entschlüsseln (zum Anzeigen)
sops --decrypt monitoring/grafana-secret.yaml

# Secret bearbeiten
sops monitoring/grafana-secret.yaml

# Flux Status prüfen
flux get all

# Cluster löschen
k3d cluster delete sops-demo
```

## Sicherheitshinweise

- **Niemals** den Private Key (`age-key.txt`) in Git committen
- Age Key sicher in 1Password oder ähnlichem speichern
- `.sops.yaml` kann öffentlich sein (enthält nur Public Keys)
- Verschlüsselte Secrets können sicher in Git gespeichert werden

## Nächste Schritte

1. Integration mit 1Password für Key Management
2. Automatisierte Key-Rotation
3. Multi-Environment Setup (dev/staging/prod)
4. CI/CD Pipeline mit SOPS Validation