# MCP Server Docker Image Release Guide

Dieses Dokument erklärt die GitHub Actions Workflow-Konfiguration für das automatische Deployen des Graphiti MCP Server Docker Images zu GitHub Container Registry (ghcr.io).

## Überblick

Der Workflow `release-mcp-server.yml` ist konfiguriert um:

- **Automatisch bei neuen Git Tags triggern** (z.B. `mcp-v1.0.0`)
- **Zwei Docker Image Varianten bauen**: standalone und combined
- **Multi-Platform Support**: Linux ARM64 und AMD64 Architekturen
- **Zu GitHub Container Registry deployen**: `ghcr.io/mfreiwald/knowledge-graph-mcp`
- **Semantische Versionierung erzwingen**: Tag muss mit `pyproject.toml` übereinstimmen

## Konfiguration

### Environment Variables (in der Workflow-Datei)

```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: mfreiwald/knowledge-graph-mcp
```

**Anpassung**: Falls du einen anderen Image-Namen/Namespace möchtest, ändere diese Werte.

### Docker Image Varianten

Der Workflow baut zwei Varianten parallel:

1. **Standalone** (`-standalone` suffix)
   - Dockerfile: `mcp_server/docker/Dockerfile.standalone`
   - Use Case: Externe Neo4j oder FalkorDB Instanzen
   - Tags: `X.Y.Z-standalone`, `latest-standalone`

2. **Combined** (keine suffix)
   - Dockerfile: `mcp_server/docker/Dockerfile`
   - Use Case: FalkorDB + MCP Server zusammen
   - Tags: `X.Y.Z`, `X.Y.Z-graphiti-A.B.C`, `latest`

## Verwendung

### Release-Workflow triggern

#### Option 1: Automatisch via Git Tag

```bash
# 1. Stelle sicher dass die Version in mcp_server/pyproject.toml aktuell ist
# 2. Erstelle und pushe einen neuen Tag
git tag mcp-v1.5.0
git push origin mcp-v1.5.0

# Der Workflow startet automatisch!
```

#### Option 2: Manuell via GitHub UI

1. Gehe zu: `GitHub Actions → Release MCP Server → Run workflow`
2. Gebe einen existierenden Tag ein (z.B. `mcp-v1.5.0`)
3. Klicke "Run workflow"

### Überprüfen des Deployments

Nach dem Workflow-Durchlauf:

```bash
# Image lokal pullen und testen
docker pull ghcr.io/mfreiwald/knowledge-graph-mcp:latest
docker pull ghcr.io/mfreiwald/knowledge-graph-mcp:latest-standalone

# Oder auf Docker Hub durchsuchen
# https://github.com/mfreiwald?tab=packages&repo_name=graphiti
```

## Workflow-Details

### Validierungen

Der Workflow führt mehrere Validierungen durch:

1. **Semantic Versioning**: Tag muss `mcp-vX.Y.Z` Format haben
2. **Version Matching**: Tag-Version muss mit `mcp_server/pyproject.toml` übereinstimmen
3. **Graphiti Core Version**: Wird automatisch von PyPI geholt und als Label gespeichert

### Build-Process

1. Repository wird checked out
2. Python 3.11 Setup für Version-Validierung
3. Docker Metadata wird generiert (Tags + Labels)
4. Multi-platform Docker Build (AMD64 + ARM64)
5. Images werden zu ghcr.io gepusht
6. Zusammenfassung wird in GitHub Step Summary angezeigt

### Container Registry Tags

Für Version `1.5.0` mit Graphiti Core `0.25.0` werden folgende Tags erstellt:

**Standalone Variant:**
- `ghcr.io/mfreiwald/knowledge-graph-mcp:1.5.0-standalone`
- `ghcr.io/mfreiwald/knowledge-graph-mcp:1.5.0-graphiti-0.25.0-standalone`
- `ghcr.io/mfreiwald/knowledge-graph-mcp:standalone` (latest tag)

**Combined Variant:**
- `ghcr.io/mfreiwald/knowledge-graph-mcp:1.5.0`
- `ghcr.io/mfreiwald/knowledge-graph-mcp:1.5.0-graphiti-0.25.0`
- `ghcr.io/mfreiwald/knowledge-graph-mcp:latest`

## Authentifizierung

### GitHub Container Registry (Standard)

Der Workflow verwendet automatisch `secrets.GITHUB_TOKEN` für ghcr.io Authentication:

```yaml
- name: Log in to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

**Keine zusätzliche Konfiguration nötig!** GitHub Actions hat standmäßig Zugriff auf ghcr.io.

### Paket-Sichtbarkeit

Standardmäßig sind Pakete auf ghcr.io private. Um sie public zu machen:

1. Gehe zu: `GitHub Settings → Packages → mfreiwald/knowledge-graph-mcp`
2. Ändere "Visibility" zu "Public"
3. Klicke "Change visibility"

## Fehlersuche

### Tag-Validierungsfehler

```
Error: Tag must follow semantic versioning: mcp-vX.Y.Z (e.g., mcp-v1.0.0)
Received: mcp-v1.5.0-beta
```

**Lösung**: Nur Release-Tags verwenden (z.B. `mcp-v1.5.0`), keine Pre-Release Tags.

### Version Mismatch

```
Error: Tag version mcp-v1.5.0 does not match mcp_server/pyproject.toml version 1.5.1
```

**Lösung**: Stelle sicher dass `mcp_server/pyproject.toml` die korrekte Version hat bevor du den Tag erstellst.

### Build-Fehler

Falls der Docker Build fehlschlägt:

1. Schaue dir die GitHub Actions Logs an
2. Führe lokal aus: `cd mcp_server && docker build -f docker/Dockerfile .`
3. Überprüfe dass alle Dockerfiles aktuell sind

## Alternative: Ohne Depot (Standard GitHub Actions)

Falls du Depot nicht verwenden möchtest oder kannst, kannst du die Standard Docker Actions verwenden. Ändere den Build-Step:

```yaml
# Original (mit Depot):
- name: Build and push Docker image (${{ matrix.variant.name }})
  uses: depot/build-push-action@v1
  with:
    project: v9jv1mlpwc
    context: ./mcp_server
    file: ./mcp_server/${{ matrix.variant.dockerfile }}
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ steps.docker_meta.outputs.tags }}
    labels: ${{ steps.docker_meta.outputs.labels }}
    ...

# Alternativ (ohne Depot - kostenlos):
- name: Build and push Docker image (${{ matrix.variant.name }})
  uses: docker/build-push-action@v5
  with:
    context: ./mcp_server
    file: ./mcp_server/${{ matrix.variant.dockerfile }}
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ steps.docker_meta.outputs.tags }}
    labels: ${{ steps.docker_meta.outputs.labels }}
    build-args: |
      MCP_SERVER_VERSION=${{ steps.version.outputs.version }}
      GRAPHITI_CORE_VERSION=${{ steps.graphiti.outputs.graphiti_version }}
      BUILD_DATE=${{ steps.meta.outputs.build_date }}
      VCS_REF=${{ steps.version.outputs.version }}
```

**Unterschiede:**
- **Depot**: Schneller (cached builds), kostenpflichtig, aber schneller als GitHub Actions runner
- **Standard Docker Actions**: Kostenlos, langsamer, aber keine Konfiguration nötig

## Nächste Schritte

1. **Test Tag erstellen**: `git tag mcp-v0.0.1 && git push origin mcp-v0.0.1`
2. **GitHub Actions ausführen lassen**
3. **Image verifizieren**: `docker pull ghcr.io/mfreiwald/knowledge-graph-mcp:0.0.1`
4. **Settings anpassen** (z.B. Paket-Sichtbarkeit)

## Referenzen

- [GitHub Container Registry Dokumentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Metadata Action](https://github.com/docker/metadata-action)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
