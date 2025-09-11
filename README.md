# SOPS Demo mit Flux GitOps

Dieses Repository demonstriert die Verwendung von SOPS (Secrets OPerationS) mit Flux GitOps für sichere Kubernetes Secret-Verwaltung.

## Voraussetzungen

- `age` Tool installiert (`brew install age`)
- `sops` Tool installiert (`brew install sops`)
- `k3d` für lokale Kubernetes Cluster (`brew install k3d`)
- `kubectl` für Kubernetes CLI (`brew install kubectl`)
- `flux` CLI für Flux GitOps (`brew install fluxcd/tap/flux`)
- `gh` CLI für GitHub (`brew install gh`)

## Komplette Step-by-Step Anleitung

### 1. Repository klonen und vorbereiten

```bash
# Repository klonen
git clone https://github.com/TimourMobi/sops-demo.git
cd sops-demo

# Age Key erstellen (wird in .gitignore ausgeschlossen)
age-keygen -o age-key.txt

# Public Key anzeigen und notieren
cat age-key.txt | grep "public key"
# Output: # public key: age18pzu5h6a7u6v73mren28s9a799lemljpfqa7py0uw76c8yh9zvnsu4eg4h
```

### 2. SOPS Konfiguration aktualisieren

```bash
# .sops.yaml mit dem neuen Public Key aktualisieren
# Ersetze den age key in .sops.yaml mit deinem Public Key
vim .sops.yaml
```

Die `.sops.yaml` sollte so aussehen:
```yaml
creation_rules:
  - path_regex: .*\.yaml$
    encrypted_regex: '^(data|stringData)$'
    age: age18pzu5h6a7u6v73mren28s9a799lemljpfqa7py0uw76c8yh9zvnsu4eg4h
```

### 3. Neues Secret mit eigenem Key verschlüsseln

```bash
# Unverschlüsseltes Secret erstellen
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

# Mit SOPS verschlüsseln (Age Key Pfad setzen)
export SOPS_AGE_KEY_FILE=age-key.txt
sops --encrypt --in-place grafana-secret-plain.yaml

# Verschlüsseltes Secret nach monitoring/ verschieben
cp grafana-secret-plain.yaml monitoring/grafana-secret.yaml

# Temporäre Datei löschen
rm grafana-secret-plain.yaml

# Änderungen committen und pushen
git add .
git commit -m "Update SOPS configuration with new Age key"
git push
```

### 4. k3d Cluster erstellen

```bash
# k3d Cluster mit Load Balancer erstellen
k3d cluster create sops-demo \
  --agents 2 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer"

# Cluster-Info anzeigen
kubectl cluster-info

# Nodes prüfen
kubectl get nodes
```

### 5. GitHub Login und Flux Bootstrap

```bash
# GitHub CLI einloggen
gh auth login

# GitHub Token für Flux exportieren
export GITHUB_TOKEN=$(gh auth token)

# Flux in den Cluster bootstrappen
flux bootstrap github \
  --owner=TimourMobi \
  --repository=sops-demo \
  --branch=main \
  --path=flux-system
```

### 6. SOPS Age Key in Kubernetes konfigurieren

```bash
# SOPS Age Key als Secret in flux-system namespace hinzufügen
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age-key.txt

# Flux Kustomization für SOPS Decryption konfigurieren
kubectl patch kustomization flux-system \
  --namespace flux-system \
  --type merge \
  --patch '{"spec":{"decryption":{"provider":"sops","secretRef":{"name":"sops-age"}}}}'
```

### 7. Deployment Status prüfen

```bash
# Flux Reconciliation Status prüfen
flux get all

# Kustomization speziell prüfen (sollte SOPS decryption zeigen)
flux get kustomizations

# Warten bis alle Resources "Ready=True" sind
# Der Output sollte zeigen:
# kustomization/flux-system    main@sha1:xxxxx    False    True    Applied revision: main@sha1:xxxxx
```

### 8. Monitoring Deployment testen

```bash
# Monitoring Namespace prüfen
kubectl get ns monitoring

# Pods in monitoring namespace prüfen
kubectl get pods -n monitoring

# Secret wurde entschlüsselt prüfen
kubectl get secrets -n monitoring

# Grafana Service prüfen
kubectl get svc -n monitoring

# HelmReleases prüfen
kubectl get helmreleases -n monitoring
```

### 9. Grafana Zugriff via Port-Forward

**WICHTIG**: Erst Port-Forward starten wenn `flux get kustomizations` zeigt dass SOPS erfolgreich entschlüsselt wurde!

```bash
# Port-Forward zu Grafana Service starten
kubectl port-forward -n monitoring svc/grafana 3000:80

# In anderem Terminal: Grafana öffnen
open http://localhost:3000
```

### 10. Grafana Login

- **URL**: http://localhost:3000
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