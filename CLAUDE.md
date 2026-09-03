# voting-app-argocd

Manifiestos de Kubernetes de la voting-app, desplegados con **ArgoCD (GitOps)**.
Este repo es la fuente de verdad: lo que está acá es lo que corre en el cluster.

## Flujo de la aplicación

`vote` escribe en `redis` → `worker` consume de redis y persiste en Postgres →
`result` lee de Postgres y muestra el resultado.

## Estructura

- `k8s/vote/`, `k8s/result/`, `k8s/worker/`, `k8s/redis/` — aplicaciones
- `k8s/postgres/` — configuración de conexión + External Secrets Operator
- `k8s/monitoring/` — Prometheus, Grafana, Loki, Promtail (namespace `monitoring`)
- `k8s/infra/argocd/` — las Applications de ArgoCD, una por componente

Namespaces: las apps van a `voting-app`, el stack de observabilidad a `monitoring`,
y las Applications viven en `argo-cd`.

## Convenciones

**Todo cambio va por Git.** Las Applications tienen `selfHeal: true` y `prune: true`.
Si hacés `kubectl edit` o `kubectl apply` a mano, ArgoCD lo revierte solo.
Para cambiar algo se modifica el manifiesto y se commitea.

**Las imágenes se actualizan solo por `newTag`.** El `image:` del Deployment queda
fijo en `:placeholder` y no se toca nunca. El pipeline corre:

    kustomize edit set image <imagen>:<sha>

que edita únicamente el `newTag` del `kustomization.yaml`. El `name` de la sección
`images` tiene que coincidir exacto con el `image:` del Deployment, sin el tag:
es la clave por la que Kustomize encuentra qué reemplazar.

**Las Applications se aplican a mano una sola vez** (bootstrap de GitOps).
A partir de ahí ArgoCD sincroniza solo.

## Comandos útiles

    kubectl config use-context minikube

    kubectl get pods -n voting-app
    kubectl get applications -n argo-cd

    # Validar el resultado de kustomize antes de commitear
    kustomize build k8s/vote

## Cosas a tener en cuenta

- **El RDS `voting-app-db` ya no existe.** `k8s/postgres/configMaps.yaml` sigue
  apuntando al endpoint viejo (`voting-app-db.cj44ye4qacfa.us-east-2.rds.amazonaws.com`),
  así que `result` y `worker` no van a conectar hasta resolverlo.
- **Las imágenes están en un ECR privado.** En minikube hacen falta credenciales
  (`aws ecr get-login-password`) o los pods quedan en `ImagePullBackOff`.
- **Los Services están configurados para EKS**: `app-vote-service` en NodePort 30080
  y `result` en NodePort 30090. Los bloques equivalentes para minikube están
  comentados dentro de los mismos archivos.
- **Ingress: pendiente.** El plan es NGINX + sslip.io con ruteo por host, en un
  directorio propio `k8s/ingress/` porque referencia Services de dos Applications
  distintas.
