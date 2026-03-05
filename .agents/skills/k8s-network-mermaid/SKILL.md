---
name: k8s-network-mermaid
description: Create or update Mermaid diagrams for Kubernetes app traffic/resource relationships (Gateway API, Ingress, or Service-only) and map each node to its defining YAML file. Use when users ask for architecture diagrams, file-to-resource traceability, namespace-scoped grouping, or parser-safe Mermaid output for README/docs.
---

# K8s Network Mermaid

1. Resolve target directory:
- If user gives an app root (for example `apps/nginx`) and manifests are not directly there, check common subdirs like `base/` and use the manifest directory you find.
- Do not assume Gateway API files exist; inspect actual files first.
2. Read manifests in this priority: `namespace.yaml`, `deployment.yaml`, `service.yaml`, `gateway.yaml`, `httproute.yaml`, `ingress.yaml`.
3. Choose the best flow that matches existing resources:
- Gateway API flow (preferred when both exist): `Client -> Gateway -> HTTPRoute -> Service -> Pod`
- Ingress flow (when `ingress.yaml` exists): `Client -> Ingress -> Service -> Pod`
- Service-only flow (when ingress/gateway resources are absent): `Client -> Service -> Pod` (and show Deployment -> Pod)
4. Label each node as `Kind/name (defined <file>.yaml)`.
5. Represent namespace boundaries with `subgraph`.
6. Keep Mermaid parser-safe:
- Use quoted labels (`["..."]`)
- End statements with `;`
- Avoid HTML tags (such as `<br/>`) and overly complex edge labels
7. Save `.mmd` next to the manifests unless the user requests another path.
8. If parse errors occur, simplify labels and punctuation first.
9. If expected files are missing, explicitly mention which files were absent and that the diagram was adapted to actual manifests.

## Templates

### Gateway API

```mermaid
flowchart LR;
CLIENT["Client host <host>"];

subgraph NS["Namespace <ns> (defined namespace.yaml)"];
  GW["Gateway/<name> (defined gateway.yaml)"];
  ROUTE["HTTPRoute/<name> (defined httproute.yaml)"];
  SVC["Service/<name> (defined service.yaml)"];
  DEP["Deployment/<name> (defined deployment.yaml)"];
  POD["Pod/<app> (from deployment.yaml)"];
end;

CLIENT -->|Host header| GW;
ROUTE -->|spec.parentRefs.name| GW;
GW -->|listeners.allowedRoutes| ROUTE;
ROUTE -->|rules.backendRefs| SVC;
SVC --> POD;
DEP --> POD;
```

### Ingress

```mermaid
flowchart LR;
CLIENT["Client host <host>"];

subgraph NS["Namespace <ns> (defined namespace.yaml)"];
  ING["Ingress/<name> (defined ingress.yaml)"];
  SVC["Service/<name> (defined service.yaml)"];
  DEP["Deployment/<name> (defined deployment.yaml)"];
  POD["Pod/<app> (from deployment.yaml)"];
end;

CLIENT -->|Host header| ING;
ING -->|spec.rules.http.paths.backend.service| SVC;
SVC --> POD;
DEP --> POD;
```

### Service-only

```mermaid
flowchart LR;
CLIENT["Client"];

subgraph NS["Namespace <ns> (defined namespace.yaml)"];
  SVC["Service/<name> (defined service.yaml)"];
  DEP["Deployment/<name> (defined deployment.yaml)"];
  POD["Pod/<app> (from deployment.yaml)"];
end;

CLIENT --> SVC;
SVC -->|spec.selector| POD;
DEP --> POD;
```

## Output Rules

- Keep diagrams minimal and directly traceable to current YAML.
- Prefer one `.mmd` file per app directory.
- If requested, also embed the same Mermaid block into README.
- Prefer file name `architecture.mmd` unless user requests another name.
