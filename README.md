# gitops-app-values

**Configuración por aplicación** del circuito **QuickTask GitOps**. Este repo es la
única fuente de verdad de _qué_ se despliega y _con qué parámetros_, mientras que el
_cómo_ (los templates de Kubernetes) vive en el chart genérico.

> Separación de responsabilidades:
> - [`helm-chart-generic-app`](https://github.com/slashispy/helm-chart-generic-app) → **cómo** se despliega (templates).
> - **`gitops-app-values`** (este repo) → **qué** se despliega y **con qué values**.
> - [`gitops-argocd-apps`](https://github.com/slashispy/gitops-argocd-apps) → **quién** reconcilia (Argo CD: App of Apps + ApplicationSet).

## Parte del circuito QuickTask GitOps

| Repo | Rol |
|---|---|
| [`quicktask`](https://github.com/slashispy/quicktask) | Aplicación: API Go + MongoDB, valida JWT, `/health` y `/ready`. |
| [`quicktask-auth`](https://github.com/slashispy/quicktask-auth) | Emisor de JWT (Quarkus, client credentials, RS256). |
| [`helm-chart-generic-app`](https://github.com/slashispy/helm-chart-generic-app) | Helm chart genérico y reutilizable (**cómo** se despliega). |
| **`gitops-app-values`** (este repo) | Values por aplicación (**qué** se despliega). |
| [`gitops-argocd-apps`](https://github.com/slashispy/gitops-argocd-apps) | App of Apps + ApplicationSet de Argo CD (**quién** reconcilia). |
| [`kind-gitops-utility-belt`](https://github.com/slashispy/kind-gitops-utility-belt) | Bootstrap del cluster KIND, tutorial y sitio del proyecto. |

📖 [Tutorial paso a paso](https://slashispy.github.io/kind-gitops-utility-belt/tutorial) · 🌐 [Sitio del proyecto](https://slashispy.github.io/kind-gitops-utility-belt/)

## Estructura

```
gitops-app-values/
└── apps/
    ├── quicktask/          # API Go + MongoDB (valida JWT)
    │   ├── metadata.yaml   # descriptor: name, namespace, chart (repoURL/path/targetRevision)
    │   └── values.yaml     # values que consumen el contrato del chart genérico
    └── quicktask-auth/     # emisor de JWT (Quarkus), mismo contrato de archivos
        ├── metadata.yaml
        └── values.yaml
```

Cada carpeta bajo `apps/<nombre>/` es una aplicación gestionada. El
`ApplicationSet` de `gitops-argocd-apps` recorre los `apps/*/metadata.yaml` con un
**git generator** y genera una `Application` de Argo CD por cada una:

- `source` del chart = `metadata.chart.repoURL` @ `metadata.chart.targetRevision`, `path`.
- `values` = el `values.yaml` hermano.
- `destination.namespace` = `metadata.namespace`.

## `metadata.yaml` (contrato)

```yaml
name: quicktask                 # nombre de la Application / release Helm
namespace: quicktask            # namespace destino
chart:
  repoURL: https://github.com/slashispy/helm-chart-generic-app
  path: .                       # raíz del repo del chart
  targetRevision: main
enabled: true                   # el ApplicationSet puede filtrar por esta clave
```

## `values.yaml`

Parametriza el chart genérico para la app. Las claves disponibles y su significado
están documentadas en el
[`values.yaml` del chart](https://github.com/slashispy/helm-chart-generic-app/blob/main/values.yaml).
Para `quicktask` se configuran: imagen `quicktask:dev`, env (`MONGO_DB_NAME`, `PORT`,
`JWT_PUBLIC_KEY_PATH`, `JWT_ISSUER`, `JWT_AUDIENCE` en ConfigMap; `MONGO_URI` en Secret),
la clave pública montada en `/keys` (`extraVolumes`/`extraVolumeMounts`), probes a
`/health` y `/ready`, y la dependencia MongoDB (`mongodb.enabled: true`).

Para `quicktask-auth` (emisor de JWT, Quarkus): imagen `quicktask-auth:dev`, env
`QUICKTASK_AUTH_*` en ConfigMap, la clave privada montada en `/keys`, el hash del
client secret leído de un Secret (`extraEnv` con `secretKeyRef`), probes a
`/q/health/live` y `/q/health/ready`, y **sin** MongoDB (`mongodb.enabled: false`):
es un servicio sin estado. Si en el futuro necesitara persistencia, usaría el MongoDB
del cluster.

> **Secretos**: las claves JWT y las credenciales del cliente de demo **no están en
> este repo**. Los values solo las referencian por nombre (`quicktask-auth-keys`,
> `quicktask-jwt-public`, `quicktask-auth-demo-client`); los crea
> `scripts/04-generate-jwt-keys.sh` (`make keys`) de `kind-gitops-utility-belt` como
> Secrets del cluster. Lo que sí queda en claro es `MONGO_URI`, un valor de ejemplo
> sin credenciales (solo para KIND). En producción se integraría External Secrets /
> SOPS / Sealed Secrets.

## Cómo agregar una nueva app o entorno

### Nueva aplicación

1. Crear `apps/<nueva-app>/`.
2. Añadir `metadata.yaml` (name, namespace, `chart.repoURL`/`path`/`targetRevision`).
3. Añadir `values.yaml` con la configuración de la app según el contrato del chart.
4. Commit + push. El `ApplicationSet` generará su `Application` automáticamente.

### Nuevo entorno (p. ej. staging/prod)

Opción recomendada: una carpeta por entorno, p. ej.
`apps/quicktask-staging/` y `apps/quicktask-prod/`, cada una con su `metadata.yaml`
(namespace y `targetRevision` propios) y su `values.yaml`.

## Verificación local

Renderizar el chart con estos values (requiere clonar también el chart):

```bash
# desde un directorio que contenga ambos repos
helm template quicktask ../helm-chart-generic-app \
  -n quicktask \
  -f apps/quicktask/values.yaml
```
