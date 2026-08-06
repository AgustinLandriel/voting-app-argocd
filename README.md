<<<<<<< HEAD
# voting-app-argocd
=======
# GitOps Voting App — EKS Deployment

Voting app desplegada en EKS con ArgoCD, RDS Postgres y ElastiCache Redis.

## Arquitectura

```
Vote (LB) → Redis (ElastiCache) → Worker → Postgres (RDS) → Result (LB)
```

| Componente | Tecnología |
|------------|------------|
| Orquestación | EKS (Kubernetes) |
| GitOps | ArgoCD |
| Base de datos | RDS PostgreSQL 16 |
| Cache / Queue | ElastiCache Redis |
| Secrets | AWS Secrets Manager + External Secrets Operator |
| Registry | ECR |
| CI | GitHub Actions |

## Estructura del repositorio

```
k8s/
├── infra/
│   ├── argocd/          # ArgoCD Applications (vote, result, worker, postgres)
│   ├── aws-keys.yaml    # Secret con credenciales AWS para ESO
│   ├── aws-secret-store.yaml  # SecretStore de External Secrets
│   └── eks-secret-store.yaml  # ClusterSecretStore (IRSA, para uso futuro)
├── postgres/
│   ├── configMaps.yaml        # Hosts de RDS y ElastiCache
│   └── external-secret.yaml   # ExternalSecret → AWS Secrets Manager
├── vote/
├── result/
└── worker/
```

## Setup inicial del cluster

### 1. Acceso al cluster EKS

El usuario IAM necesita estar en el trust policy del rol `eks-admin`:

```bash
# Asumir el rol eks-admin
CREDS=$(aws sts assume-role --role-arn arn:aws:iam::325503636955:role/eks-admin --role-session-name eks-session)
export AWS_ACCESS_KEY_ID=$(echo $CREDS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Credentials']['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Credentials']['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo $CREDS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Credentials']['SessionToken'])")

# Configurar kubeconfig
aws eks update-kubeconfig --region us-east-2 --name voting-app
```

> El token dura 1 hora. Repetir el assume-role cuando expire.

### 2. Instalar ArgoCD

```bash
kubectl create namespace argo-cd
kubectl apply -n argo-cd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Corrección necesaria post-instalación** — los ClusterRoleBindings del installer apuntan al namespace `argocd` pero instalamos en `argo-cd`:

```bash
for crb in argocd-application-controller argocd-applicationset-controller argocd-server; do
  kubectl patch clusterrolebinding $crb --type=json \
    -p='[{"op":"replace","path":"/subjects/0/namespace","value":"argo-cd"}]'
done

# Permisos para listar CronJobs (necesario para sync)
kubectl patch clusterrole argocd-application-controller --type=json \
  -p='[{"op":"add","path":"/rules/-","value":{"apiGroups":["batch"],"resources":["cronjobs","jobs"],"verbs":["get","list","watch"]}}]'

kubectl rollout restart statefulset/argocd-application-controller -n argo-cd
```

### 3. Crear namespace y aplicar ArgoCD Applications

```bash
kubectl create namespace voting-app
kubectl apply -f k8s/infra/argocd/
```

### 4. Instalar External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --wait
```

Aplicar el SecretStore y los recursos necesarios en el namespace de la app:

```bash
kubectl apply -f k8s/infra/aws-keys.yaml -n voting-app
kubectl apply -f k8s/infra/aws-secret-store.yaml -n voting-app
```

ArgoCD sincroniza automáticamente el `ExternalSecret` de postgres desde el repo.

### 5. Acceder a ArgoCD UI

```bash
# Port-forward (requiere tener las credenciales de eks-admin exportadas)
kubectl port-forward svc/argocd-server -n argo-cd 8080:80
```

- URL: `http://localhost:8080`
- Usuario: `admin`
- Contraseña: `kubectl get secret argocd-initial-admin-secret -n argo-cd -o jsonpath="{.data.password}" | base64 -d`

## Endpoints de la app

| Servicio | URL |
|----------|-----|
| Vote | `http://ae0a40f930cdb490fb14324af842989e-2106747957.us-east-2.elb.amazonaws.com` |
| Result | `http://a8b79d1317bcc4a8e99acae79cef8d24-92088127.us-east-2.elb.amazonaws.com:3000` |

## Secreto en AWS Secrets Manager

El secreto `voting-app` contiene:

```json
{
  "POSTGRES_USER": "postgres",
  "POSTGRES_PASSWORD": "...",
  "POSTGRES_DB": "votes"
}
```

El `ExternalSecret` en `k8s/postgres/external-secret.yaml` lo lee y crea el Secret de Kubernetes `postgres-secret` automáticamente.

## Configuración de red (Security Groups)

Los Security Groups de RDS y ElastiCache deben permitir tráfico desde el SG de los nodos EKS:

| Recurso | Puerto | SG origen |
|---------|--------|-----------|
| RDS Postgres | 5432 | SG nodos EKS |
| ElastiCache Redis | 6379 | SG nodos EKS |

> Al recrear el node group se asigna un nuevo SG. Verificar que las reglas apunten al SG correcto con `aws ec2 describe-instances --filters "Name=tag:aws:eks:cluster-name,Values=voting-app"`.

## Configuración de RDS

RDS usa un parameter group custom (`voting-app-postgres16`) con `rds.force_ssl = 0` para permitir conexiones sin SSL desde los pods.

## Node group

Las imágenes ECR son `amd64`. El node group debe usar instancias x86_64:

- Tipo actual: `t3.small` (`AL2023_x86_64_STANDARD`)
- Node group: `standard-workers-amd64`

> Si se usa ARM (Graviton / t4g), las imágenes deben ser multi-arch o recompiladas para `arm64`.

## Flujo GitOps

1. El pipeline de GitHub Actions buildea las imágenes y las pushea a ECR
2. Actualiza los tags de imagen en este repo (`k8s/vote/`, `k8s/result/`, `k8s/worker/`)
3. ArgoCD detecta el cambio y sincroniza automáticamente al cluster
>>>>>>> 6179d1c (ordenamiento de yamls)
