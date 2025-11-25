# Cilium Network & Gateway Addon для Talos Kubernetes

- [Cilium Network \& Gateway Addon для Talos Kubernetes](#cilium-network--gateway-addon-для-talos-kubernetes)
  - [1. Обзор](#1-обзор)
  - [2. Требования](#2-требования)
  - [3. Структура аддона](#3-структура-аддона)
  - [4. Конфигурационные переменные](#4-конфигурационные-переменные)
    - [4.1. Обязательные переменные](#41-обязательные-переменные)
    - [4.2. Секреты](#42-секреты)
  - [5. Развертывание](#5-развертывание)
    - [5.1. Установка Cilium](#51-установка-cilium)
    - [5.2. Настройка внешних IP](#52-настройка-внешних-ip)
    - [5.3. L2 Announcement](#53-l2-announcement)
    - [5.4. Shared Gateway](#54-shared-gateway)
  - [6. Примеры использования](#6-примеры-использования)
    - [6.1. Создание сервиса с внешним IP](#61-создание-сервиса-с-внешним-ip)
    - [6.2. Маршрутизация через Gateway API](#62-маршрутизация-через-gateway-api)
  - [7. Валидация установки](#7-валидация-установки)
    - [7.1. Проверка состояния Cilium](#71-проверка-состояния-cilium)
    - [7.2. Проверка Gateway API](#72-проверка-gateway-api)
  - [8. Troubleshooting](#8-troubleshooting)
    - [8.1. Распространенные проблемы](#81-распространенные-проблемы)
    - [8.2. Полезные команды](#82-полезные-команды)
  - [9. Особенности production-эксплуатации](#9-особенности-production-эксплуатации)

## 1. Обзор

Аддон `talos-cilium` предоставляет полный сетевой стек для Kubernetes-кластера на базе Talos Linux с использованием Cilium в качестве CNI (Container Network Interface). Реализует enterprise-grade функционал для сетевой безопасности, балансировки нагрузки и маршрутизации трафика с использованием стандартов Kubernetes Gateway API.

**Ключевые возможности:**

- ✅ Полная замена kube-proxy с аппаратным ускорением (XDP)
- ✅ L7 маршрутизация и TLS termination через Gateway API
- ✅ Внешние IP-адреса для сервисов типа LoadBalancer
- ✅ Политики безопасности на уровне L3/L4/L7
- ✅ Мониторинг и observability через Hubble UI
- ✅ OpenTelemetry интеграция
- ✅ Multi-zone и multi-cluster поддержка

## 2. Требования

| Компонент         | Версия/Требование      | Примечание                            |
| ----------------- | ---------------------- | ------------------------------------- |
| Talos Cluster     | >= 1.7                 | Базовый кластер должен быть развернут |
| Helm              | >= 3.8                 | Для установки Cilium                  |
| Gateway API CRDs  | v1.0.0+                | Устанавливаются автоматически         |
| Внешние IP-адреса | Выделенный диапазон    | Для сервисов типа LoadBalancer        |
| TLS-сертификаты   | Для доменов корпорации | Для TLS termination на шлюзе          |

## 3. Структура аддона

```
talos-cilium/
├── ansible/
│   ├── pre_tasks.yml           # Предварительные задачи (установка Cilium)
│   ├── wait_tasks.yml          # Проверки после установки
│   └── <cluster-name>.yml      # Кластерные переменные
├── manifests-to-apply/
│   ├── loadbalancer-config.yaml  # Настройка IPAM и L2 announcement
│   ├── shared-gateway-tls.yaml   # TLS-сертификаты для шлюза
│   └── 3-shared-gateway.yaml     # Конфигурация Gateway API
└── tests/                      # Тестовые манифесты для валидации
```

## 4. Конфигурационные переменные

### 4.1. Обязательные переменные

| Имя переменной                   | Описание                      | Пример значения |
| -------------------------------- | ----------------------------- | --------------- |
| `talos_cilium_network`           | CIDR для подов в кластере     | `10.224.0.0/13` |
| `talos_cilium_node_network_size` | Размер подсети на ноду        | `22`            |
| `talos_loadbalancer_start_ip`    | Начальный IP для LoadBalancer | `10.10.61.50`   |
| `talos_loadbalancer_end_ip`      | Конечный IP для LoadBalancer  | `10.10.61.59`   |
| `talos_shared_gateway_ip`        | IP-адрес для shared gateway   | `10.10.61.51`   |

### 4.2. Секреты

| Имя переменной                  | Описание                    | Формат                         |
| ------------------------------- | --------------------------- | ------------------------------ |
| `talos_shared_gateway_corp_key` | TLS-ключ для домена `.corp` | Зашифрован через ansible-vault |
| `talos_shared_gateway_com_key`  | TLS-ключ для домена `.com`  | Зашифрован через ansible-vault |

## 5. Развертывание

### 5.1. Установка Cilium

Аддон использует Helm для установки Cilium версии 1.18.2 со следующими ключевыми настройками:

```yaml
# Ключевые параметры Helm
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList: [10.224.0.0/13]
    clusterPoolIPv4MaskSize: 22
ipv4NativeRoutingCIDR: 10.224.0.0/13
kubeProxyReplacement: true
loadBalancer:
  mode: dsr
  acceleration: native
routingMode: native
autoDirectNodeRoutes: true
l2announcements:
  enabled: true
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
gatewayAPI:
  enabled: true
```

### 5.2. Настройка внешних IP

Создается пул IP-адресов для сервисов типа LoadBalancer:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "first-pool"
spec:
  blocks:
    - start: "10.10.61.50"
      stop: "10.10.61.59"
```

### 5.3. L2 Announcement

Автоматическая анонсация внешних IP в локальной сети:

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy1
spec:
  serviceSelector:
    matchLabels:
      io.cilium/lb-enabled: "true"
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  interfaces:
    - ^eth[0-9]+
  externalIPs: true
  loadBalancerIPs: true
```

### 5.4. Shared Gateway

Конфигурация многофункционального шлюза с поддержкой:

- HTTP/HTTPS для доменов `.corp` и `.com`
- TLS termination с автоматическим редиректом с HTTP на HTTPS
- L7 маршрутизация на основе хостов
- Балансировка нагрузки между сервисами

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: cilium-gateway
spec:
  infrastructure:
    annotations:
      lbipam.cilium.io/ips: "10.10.61.51"
  listeners:
    - name: https-corp
      protocol: HTTPS
      port: 443
      hostname: "*.okbtsp.corp"
      tls:
        mode: Terminate
        certificateRefs:
          - name: gateway-cert-corp
    - name: http-corp # Автоматический редирект на HTTPS
      # ...конфигурация...
```

## 6. Примеры использования

### 6.1. Создание сервиса с внешним IP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  annotations:
    io.cilium/lb-ipam-ips: "10.10.61.52" # Статический IP из пула
  labels:
    io.cilium/lb-enabled: "true" # Требуется для L2 announcement
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### 6.2. Маршрутизация через Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
spec:
  hostnames:
    - "app.okbtsp.corp"
  parentRefs:
    - name: shared-gateway
      namespace: cilium-gateway
      sectionName: https-corp
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-service
          port: 80
```

## 7. Валидация установки

### 7.1. Проверка состояния Cilium

```bash
# Проверка состояния агентов
talosctl kubectl get pods -n kube-system -l k8s-app=cilium

# Проверка оператора
talosctl kubectl get deployment -n kube-system cilium-operator

# Проверка Hubble
talosctl kubectl get pods -n kube-system -l k8s-app=hubble
```

### 7.2. Проверка Gateway API

```bash
# Проверка GatewayClass
talosctl kubectl get gatewayclass

# Проверка Gateway
talosctl kubectl get gateway -n cilium-gateway

# Проверка HTTPRoute
talosctl kubectl get httproute
```

## 8. Troubleshooting

### 8.1. Распространенные проблемы

| Проблема                                    | Решение                                                                             |
| ------------------------------------------- | ----------------------------------------------------------------------------------- |
| Сервис не получает внешний IP               | Проверить наличие аннотации `io.cilium/lb-ipam-ips` и лейбла `io.cilium/lb-enabled` |
| TLS сертификаты не применяются              | Проверить корректность формата секретов в namespace `cilium-gateway`                |
| Gateway не переходит в состояние Programmed | Проверить логи cilium-operator и gateway-controller                                 |
| Отсутствует соединение между подами         | Проверить состояние Cilium агентов на всех нодах                                    |

### 8.2. Полезные команды

```bash
# Логи Cilium агента на конкретной ноде
talosctl kubectl logs -n kube-system -l k8s-app=cilium --tail=100

# Диагностика сети
talosctl kubectl exec -n kube-system cilium-xxxxx -- cilium status

# Просмотр маршрутов Hubble
talosctl kubectl exec -n kube-system hubble-relay-xxxxx -- hubble observe
```

## 9. Особенности production-эксплуатации

- **Высокая доступность:** Cilium оператор запускается в 3 репликах на controlplane/infra нодах
- **Zero-downtime обновления:** Используется автоматический rollout restart с аннотациями
- **Сетевая безопасность:** Все компоненты работают с минимальными необходимыми привилегиями
- **Мониторинг:** Интеграция с OpenTelemetry для сбора метрик и трейсов
- **Сертификаты:** TLS-сертификаты для шлюза хранятся в зашифрованном виде через ansible-vault
- **Важно:** В примерах выше не используются NetworkPolicy для упрощения, но в production-средах обязательно настройте CiliumNetworkPolicy/EgressGatewayPolicy для сегментации трафика и защиты от боковых атак в соответствии с принципом минимальных привилегий.
