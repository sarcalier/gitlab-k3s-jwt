# Порядок развёртывания ролей

## Правильный порядок запуска playbooks:

### 1. Базовая инфраструктура
```bash
# 1. Установка k3s
ansible-playbook playbooks/install-k3s.yml

# 2. Установка GitLab
ansible-playbook playbooks/install-gitlab.yml

# 3. Установка Keycloak
ansible-playbook playbooks/install-keycloak.yml
```

### 2. Конфигурация Keycloak
```bash
# 4. Настройка Keycloak (realm, client, user, group)
ansible-playbook playbooks/configure-keycloak.yml
```

### 3. Интеграция GitLab с Keycloak
```bash
# 5. Настройка GitLab OIDC с Keycloak (ОБЯЗАТЕЛЬНО перед k3s-gitlab-jwt!)
ansible-playbook playbooks/configure-gitlab-oidc.yml
```

### 4. Интеграция k3s с Keycloak
```bash
# 6. Настройка k3s OIDC с Keycloak (для прямых JWT токенов)
ansible-playbook playbooks/configure-k3s-oidc.yml
```

### 5. Интеграция k3s с GitLab JWT
```bash
# 7. Настройка k3s для приёма GitLab CI_JOB_JWT (требует gitlab-oidc!)
ansible-playbook playbooks/configure-k3s-gitlab-jwt.yml
```

## Важно:

- **`configure-gitlab-oidc.yml`** ДОЛЖЕН запускаться ПЕРЕД `configure-k3s-gitlab-jwt.yml`
- Причина: GitLab должен быть настроен на синхронизацию групп из Keycloak
- Без этого группы не будут доступны в GitLab JWT токенах

## Исправление проблемы с issuer:

Если после изменения Keycloak на HTTPS issuer сломался логин в GitLab:

```bash
# Перезапустить роль gitlab-oidc для обновления конфигурации
ansible-playbook playbooks/configure-gitlab-oidc.yml
```

Это обновит OIDC secret в GitLab с правильным HTTPS issuer.
