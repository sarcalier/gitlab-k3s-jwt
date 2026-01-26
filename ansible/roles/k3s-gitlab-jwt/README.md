# k3s-gitlab-jwt Role

Настраивает k3s для приема JWT токенов от GitLab CI/CD и Keycloak через структурированную конфигурацию аутентификации.

## Возможности

- ✅ Поддержка множественных OIDC issuer'ов (GitLab + Keycloak)
- ✅ Использование структурированной конфигурации (`--authentication-config`)
- ✅ Автоматическая миграция со старых флагов командной строки
- ✅ Автоматическое получение CA сертификатов

## Issuer'ы

### 1. GitLab (CI/CD JWT)
- URL: `https://gitlab.gitlab.local`
- Используется для: `CI_JOB_JWT` токенов в пайплайнах
- Username prefix: `gitlab:`
- Groups prefix: `gitlab:`

### 2. Keycloak (Direct Auth)
- URL: `https://keycloak.local/realms/gitlab`
- Используется для: прямой аутентификации пользователей
- Username prefix: настраивается (по умолчанию: `""`)
- Groups prefix: настраивается (по умолчанию: `""`)

## Файлы

- `/etc/k3s-gitlab-jwt/authentication-config.yaml` - структурированная конфигурация
- `/etc/k3s-gitlab-jwt/gitlab-ca.crt` - CA сертификат GitLab
- `/etc/k3s-gitlab-jwt/gitlab-rbac.yaml` - RBAC манифесты
- `/etc/k3s-gitlab-jwt/gitlab-ci-jwt.yml` - шаблон `.gitlab-ci.yml`

## Переменные

См. `defaults/main.yml` для полного списка переменных.

### GitLab
- `k3s_gitlab_url`
- `k3s_gitlab_client_id`
- `k3s_gitlab_username_claim`
- `k3s_gitlab_groups_claim`

### Keycloak
- `k3s_oidc_issuer_url`
- `k3s_oidc_client_id`
- `k3s_oidc_username_claim`
- `k3s_oidc_groups_claim`
- `k3s_oidc_username_prefix`
- `k3s_oidc_groups_prefix`

## Зависимости

- Роль `k3s-oidc` (опционально) - для Keycloak CA сертификата
- Если Keycloak CA не существует, issuer все равно добавляется без `certificateAuthority`

## Миграция

Роль автоматически:
1. Обнаруживает старые `oidc-*` флаги
2. Удаляет их из `config.yaml`
3. Создает структурированную конфигурацию
4. Добавляет `--authentication-config` в k3s config
