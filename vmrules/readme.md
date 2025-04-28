
```yaml
# === Service Account ===
serviceAccount:
  create: false  # Используем существующий ServiceAccount (рекомендуется создать отдельный для production)

# === Основные параметры сервера ===
server:
  nameOverride: vmrules       # Короткое имя ресурсов
  fullnameOverride: vmrules   # Фиксированное полное имя
  replicaCount: 2             # 2 реплики для отказоустойчивости (минимум для HA)

  # === Настройки подключения к данным ===
  datasource:
    url: "http://vmselect.monitoring.svc.cluster.local:8481/select/0/prometheus/"  # Источник метрик
  
  remote:
    write:
      url: "http://vminsert.monitoring.svc.cluster.local:8480/insert/0/prometheus/"  # Куда писать результаты вычислений
  
  # === Интеграция с Alertmanager ===
  notifier:
    alertmanager:
      url: "http://alertmanager.monitoring.svc.cluster.local:9093"  # Эндпоинт Alertmanager
      # Для production рекомендуется добавить:
      # - basic_auth: при необходимости
      # - tls_config: для безопасного соединения

  # === Дополнительные параметры ===
  extraArgs:
    rule: 
      - /rules/*/*.yaml  # Шаблон поиска файлов правил

  # === Распределение подов ===
  affinity:
    podAntiAffinity:  
      requiredDuringSchedulingIgnoredDuringExecution:  # Жесткое требование
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: [vmrules]
          topologyKey: kubernetes.io/hostname  # Разные физические ноды

  # === Ресурсы (требуют увеличения для production) ===
  resources:
    requests:
      cpu: 15m     # Абсолютный минимум (рекомендуется 100m+)
      memory: 25Mi  # Слишком мало (рекомендуется 128Mi+)
    limits:
      cpu: 30m     # Лимит (рекомендуется 500m+)
      memory: 100Mi # Лимит (рекомендуется 256Mi+)

  # === Подключение правил из ConfigMap ===
  extraVolumes:
    - name: alertmanager-rules  # Правила для мониторинга Alertmanager
      configMap:
        name: alertmanager-rules
        # Для production добавить:
        # items: для точечного подключения файлов
        # defaultMode: 0444  # Права только на чтение

    - name: node-rules      # Правила для node-экспортера
      configMap:
        name: node-rules

    - name: kubernetes-rules  # Правила для Kubernetes-компонентов
      configMap:
        name: kubernetes-rules

    - name: minio-rules     # Правила для MinIO
      configMap:
        name: minio-rules

  extraVolumeMounts:
    - name: alertmanager-rules
      mountPath: /rules/alertmanager-rules
      readOnly: true  # Рекомендуется для security

    - name: node-rules
      mountPath: /rules/node-rules
      readOnly: true

    - name: kubernetes-rules
      mountPath: /rules/kubernetes-rules
      readOnly: true

    - name: minio-rules
      mountPath: /rules/minio-rules
      readOnly: true

  # === Сервис ===
  service:
    enabled: true
    annotations:
      prometheus.io/scrape: "true"  # Разрешить сбор метрик
      prometheus.io/port: "8880"    # Порт метрик
      # Для production добавить:
      # prometheus.io/scheme: "http"  # Явное указание схемы

  # === Ingress ===
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer  # Автовыпуск сертификатов
      # Дополнительные аннотации для production:
      # nginx.ingress.kubernetes.io/auth-type: basic
      # nginx.ingress.kubernetes.io/auth-secret: basic-auth
    hosts:
    - name: vmrules.dev.local  # Домен (заменить на production)
      path: /
      port: 8880
    tls:
    - secretName: vmalert-tls
      hosts:
        - vmrules.dev.local

# === Alertmanager ===
alertmanager:
  enabled: false  # Используется внешний Alertmanager
```


**Проверка работоспособности**:
```bash
# Проверить загруженные правила
curl http://vmrules:8880/api/v1/rules

# Проверить соединение с Alertmanager
curl http://vmrules:8880/api/v1/alertmanagers

# Проверить метрики
curl http://vmrules:8880/metrics
```


