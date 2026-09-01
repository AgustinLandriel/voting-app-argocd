# voting-app-argocd

Manifiestos de Kubernetes de la **voting app**, sincronizados al cluster por **ArgoCD**. Este repo es la fuente de verdad del estado deseado: nada se aplica a mano, todo cambio entra por un commit.

La app es un clásico de tres servicios — se vota en uno, un worker procesa la cola y un tercero muestra los resultados en vivo — usado acá como excusa para armar una plataforma completa de punta a punta: infraestructura como código, CI/CD, GitOps, gestión de secretos y observabilidad.

## Los tres repos

| Repo | Qué hay adentro |
|------|-----------------|
| **[voting-app-local](https://github.com/AgustinLandriel/voting-app-local)** | El código fuente de las tres apps, sus Dockerfiles, tests y la configuración de análisis estático |
| **[voting-app-tf](https://github.com/AgustinLandriel/voting-app-tf)** | La infraestructura en Terraform: VPC, RDS, security groups, el bucket de estado, ECR y un cluster K3s sobre EC2 |
| **voting-app-argocd** (este) | Los manifiestos de Kubernetes que ArgoCD observa y sincroniza |

## Qué se construyó

**GitOps con ArgoCD.** Seis `Application`, una por componente, con `prune` y `selfHeal` activados: si alguien toca algo directo en el cluster, ArgoCD lo detecta como drift y lo revierte. El pipeline nunca hace `kubectl apply` — solo actualiza el tag de imagen en este repo y ArgoCD se encarga del resto.

**Kustomize** en cada directorio, para que el pipeline actualice únicamente el `newTag` con `kustomize edit set image` y los manifiestos no se toquen nunca más. Los dashboards de Grafana se generan como ConfigMaps con `configMapGenerator`, leyendo los `.json` sueltos del repo.

**Secretos fuera de Git.** Las credenciales viven en AWS Secrets Manager. El External Secrets Operator las lee mediante un `SecretStore` y crea el `Secret` de Kubernetes solo, con refresco periódico. En el repo no hay una sola credencial.

**Observabilidad completa**, desplegada con manifiestos propios en vez de charts empaquetados, para entender cada pieza:

- **Prometheus** con service discovery vía la API de Kubernetes y su propio RBAC, descubriendo targets por anotaciones en los Services
- **Grafana** con datasource y dashboards provisionados desde Git — uno de negocio, uno técnico y uno de logs
- **Loki + Promtail** para agregación centralizada de logs, con Promtail como DaemonSet

**Salud de los pods** declarada en todos los workloads: `readinessProbe` para que ningún pod reciba tráfico antes de estar listo, `livenessProbe` para reiniciar los que quedan colgados, `initContainers` para esperar dependencias, y `resources` con requests y limits en cada contenedor.

## Tecnologías

| Capa | Stack |
|------|-------|
| Apps | Python/Flask (vote), Node.js (worker y result) |
| Orquestación | Kubernetes — minikube en local, EKS y K3s en AWS |
| GitOps | ArgoCD |
| Manifiestos | Kustomize |
| Datos | PostgreSQL (RDS) y Redis como cola |
| Secretos | AWS Secrets Manager + External Secrets Operator |
| Observabilidad | Prometheus, Grafana, Loki, Promtail |
| Registry | Amazon ECR |
| CI/CD | GitHub Actions, autenticado contra AWS por OIDC |
| Infraestructura | Terraform |

## Estructura

```
k8s/
├── infra/argocd/     # las Application de ArgoCD, una por componente
├── vote/             # front de votación (Python/Flask)
├── result/           # tablero de resultados (Node.js)
├── worker/           # procesador de la cola (Node.js)
├── redis/            # cola de votos
├── postgres/         # SecretStore, ExternalSecret y configuración de conexión
└── monitoring/       # Prometheus, Grafana, Loki y Promtail
```
