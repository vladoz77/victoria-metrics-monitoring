## Создаем секрет из файлов сертификата
```bash
kubectl create secret generic etcd-ca \
--from-file /etc/kubernetes/pki/etcd/server.crt \
--from-file /etc/kubernetes/pki/etcd/server.key \
-n monitoring
```


## О конфигурации
```yaml
# === Базовая конфигурация ===
replicaCount: 3                     # 3 реплики для отказоустойчивости
mode: statefulSet                   # Использование StatefulSet вместо Deployment
statefulSet:
  clusterMode: true                 # Включение кластерного режима
  replicationFactor: 1              # Фактор репликации данных между экземплярами

# === Идентификация ===
nameOverride: vmagent               # Базовое имя ресурсов
fullnameOverride: vmagent           # Фиксированное полное имя
annotations:
  configmap.reloader.stakater.com/reload: vmagent-config  # Автоперезагрузка при изменении конфига

# === Параметры работы ===
extraArgs:
  promscrape.cluster.memberLabel: vmagent_instance  # Метка для идентификации в кластере

# === Ресурсы ===
resources:
  limits:
    cpu: 100m      # Лимит: 0.1 ядра CPU
    memory: 200Mi  # Лимит: 200 MiB RAM
  requests:
    cpu: 20m       # Гарантировано: 0.02 ядра CPU
    memory: 100Mi  # Гарантировано: 100 MiB RAM

# === TLS-сертификаты ===
extraVolumeMounts:
- name: etcd-tls-volume            # Монтирование сертификатов etcd
  mountPath: /tls/etcd-tls

extraVolumes:
- name: etcd-tls-volume
  secret:
    secretName: etcd-ca            # Secret с TLS-сертификатами

# === Направление метрик ===
remoteWrite:
  - url: "http://vminsert.monitoring.svc.cluster.local:8480/insert/0/prometheus/"  # Куда отправлять метрики

# === Сетевые настройки ===
service:
  enabled: true
  annotations:
    prometheus.io/scrape: "true"   # Разрешить сбор метрик самого vmagent
    prometheus.io/port: "8429"     # Порт метрик vmagent

ingress:
  enabled: true                    # Включение Ingress
  annotations:
    cert-manager.io/cluster-issuer: ca-issuer  # Автоматическая выдача сертификатов
  hosts:
  - name: vmagent.dev.local        # Домен для доступа
    path: /
    port: http
  tls:
  - secretName: vmagent-tls        # TLS-сертификат
    hosts:
    - vmagent.dev.local

# === Распределение подов ===
affinity:
  podAntiAffinity:                 # Жесткое требование распределения
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [vmagent]
        topologyKey: kubernetes.io/hostname  # Разные физические ноды

# === Хранение данных ===
persistence:
  enabled: true                    # Включение постоянного хранилища
  size: 1Gi                        # Размер тома (мало для production)
  storageClassName: longhorn-data  # Тип хранилища

# === Конфигурация сбора метрик ===
config:
  global:
    scrape_interval: 10s           # Интервал сбора метрик по умолчанию

  scrape_configs:
    # Конфигурация сбора метрик etcd
    - job_name: etcd
      honor_labels: true           # Сохранять оригинальные метки
      kubernetes_sd_configs:       # Автообнаружение в Kubernetes
      - role: pod
        namespaces:
          names:
          - kube-system
      scheme: https                # Использовать HTTPS
      tls_config:                 # Настройки TLS
        insecure_skip_verify: true # Не проверять сертификат (не для production!)
        cert_file: /tls/etcd-tls/server.crt  # Сертификат
        key_file: /tls/etcd-tls/server.key   # Ключ
      relabel_configs:             # Правила преобразования меток
      - source_labels: [__meta_kubernetes_pod_label_component]
        regex: etcd
        action: keep               # Оставлять только etcd
      # Дополнительные правила преобразования меток...
      
    # Аналогичные конфигурации для других компонентов:
    # - apiserver
    # - kubelet
    # - cadvisor
    # - kube-proxy
    # - kube-controller-manager
    # - kube-scheduler
    # - calico
    # - Автообнаружение endpoints/pods по аннотациям
```

### Ключевые особенности:

1. **Кластерный режим**:
   - 3 реплики vmagent в StatefulSet
   - Распределение по разным нодам (podAntiAffinity)
   - Настройка кластерного режима сбора метрик

2. **Безопасность**:
   - Использование TLS для etcd
   - Ingress с автоматической выдачей сертификатов (cert-manager)
   - Отдельный volume для сертификатов

3. **Гибкость сбора метрик**:
   - Преднастроенные конфигурации для всех ключевых компонентов Kubernetes
   - Поддержка автообнаружения по аннотациям
   - Гибкие правила преобразования меток (relabel_configs)

4. **Надежность**:
   - Настройка remoteWrite в vminsert
   - Постоянное хранилище для данных vmagent


