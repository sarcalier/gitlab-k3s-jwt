# Ручная настройка GitLab CI_JOB_JWT для k3s

## Как работает

```
GitLab CI/CD → CI_JOB_JWT (с groups_direct) → k3s API → Kubernetes RBAC
```

1. GitLab генерирует JWT токен для пайплайна с `groups_direct: ["k8s-deployers"]` или любое другое название группы
2. k3s проверяет токен через JWT от GitLab
3. k3s извлекает группы: `gitlab:k8s-deployers`
4. Kubernetes RBAC проверяет права на основе группы (можно настроить и на основе индивидальных учеток из gitlab)

---

## Ручная настройка

### 1. Настроить k3s для GitLab OIDC

```bash
# Добавить в /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "oidc-issuer-url=https://gitlab.gitlab.local"
  - "oidc-client-id=https://gitlab.gitlab.local"
  - "oidc-username-claim=user_email"
  - "oidc-groups-claim=groups_direct"
  - "oidc-username-prefix=gitlab:"
  - "oidc-groups-prefix=gitlab:"
  - "oidc-required-claim=iss:https://gitlab.gitlab.local"
  - "oidc-ca-file=/etc/k3s-gitlab-jwt/gitlab-ca.crt"

# Получить SSL сертификат GitLab
mkdir -p /etc/k3s-gitlab-jwt
openssl s_client -showcerts -connect gitlab.gitlab.local:443 </dev/null 2>/dev/null | \
  openssl x509 -outform PEM > /etc/k3s-gitlab-jwt/gitlab-ca.crt

# Перезапустить k3s
systemctl restart k3s
```

### 2. Создать RBAC в Kubernetes

```bash
# Создать namespace
kubectl create namespace app-deploy

# Создать Role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-deployer
  namespace: app-deploy
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

# Создать RoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-deployer-binding
  namespace: app-deploy
subjects:
- kind: Group
  name: gitlab:k8s-deployers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: gitlab-deployer
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 3. Настроить GitLab

```bash
# Включить feature flag (GitLab 17.3+)
kubectl exec -it deployment/gitlab-webservice-default -n gitlab -- bash
cd /srv/gitlab
gitlab-rails console
Feature.enable(:ci_jwt_groups_direct)
exit
```

### 4. Создать группу в GitLab через UI


### 5. Настроить пайплайн

```yaml
# .gitlab-ci.yml
variables:
  K8S_API_SERVER: "https://192.168.1.244:6443"
  DEPLOY_NAMESPACE: "app-deploy"
  GIT_SSL_NO_VERIFY: "true"

.kube_auth:
  id_tokens:
    KUBE_TOKEN:
      aud: "https://gitlab.gitlab.local"
  before_script:
    - kubectl config set-cluster k3s --server="${K8S_API_SERVER}" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab-jwt --token="${KUBE_TOKEN}"
    - kubectl config set-context gitlab-deploy --cluster=k3s --user=gitlab-jwt --namespace="${DEPLOY_NAMESPACE}"
    - kubectl config use-context gitlab-deploy

deploy:
  extends: .kube_auth
  image: bitnami/kubectl:latest
  script:
    - kubectl get pods -n ${DEPLOY_NAMESPACE}
```

---

## Проверка

```bash
# В пайплайне проверить токен
echo "$KUBE_TOKEN" | cut -d'.' -f2 | base64 -d | jq '.groups_direct'
# Должно показать: ["k8s-deployers"]

```

---

## Требования

- GitLab с OIDC от Keycloak (группы gitlab необходимо создать вручную)
- Группа `k8s-deployers` создана в GitLab
- Пользователь в группе `k8s-deployers` в GitLab
- Feature flag `ci_jwt_groups_direct` включен (GitLab 17.3+)
