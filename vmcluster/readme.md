Вот детальное описание вашей конфигурации кластера VictoriaMetrics в стиле аналогичном описанию vmsingle, но для кластерного режима:

```yaml
# === Глобальные настройки ===
rbac:
  create: false         # Использовать существующие RBAC-роли
serviceAccount:
  create: false         # Использовать существующий ServiceAccount

# === vmselect ===
# Компонент для выполнения запросов (аналог читающего узла БД)
vmselect:
  enabled: true         # Включение компонента
  replicaCount: 3       # 3 реплики для отказоустойчивости
  fullnameOverride: vmselect  # Фиксированное имя ресурсов
  
  # Параметры работы:
  extraArgs:
    replicationFactor: 3  # При выполнении запросов через vmselect, система выбирает одну из доступных копий данных (обычно самую ближайшую или наиболее доступную).
    dedup.minScrapeInterval: 10s  # Дедупликация метрик в 10-секундном окне

  # Распределение подов:
  affinity:
    podAntiAffinity:  
      requiredDuringSchedulingIgnoredDuringExecution:  # Жесткое требование
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [vmselect]
        topologyKey: kubernetes.io/hostname  # Разные физические ноды

  # Ресурсы (минимальные):
  resources:
    limits:
      cpu: 100m       # Лимит: 0.1 ядра CPU
      memory: 200Mi   # Лимит: 200 MiB RAM
    requests:
      cpu: 50m        # Гарантировано: 0.05 ядра CPU
      memory: 100Mi   # Гарантировано: 100 MiB RAM

  # Мониторинг:
  podAnnotations:
    prometheus.io/scrape: "true"  # Разрешить сбор метрик
    prometheus.io/port: "8481"    # Порт метрик vmselect

# === vminsert ===
# Компонент для приема метрик (аналог пишущего узла БД)
vminsert:
  enabled: true         # Включение компонента
  replicaCount: 3       # 3 реплики для балансировки нагрузки
  fullnameOverride: vminsert  # Фиксированное имя ресурсов
  
  # Параметры работы:
  extraArgs:
    replicationFactor: 3  # Когда данные поступают в vminsert, он направляет их в три разных пода vmstorage (так как replicationFactor равен 3).

  # Распределение подов (аналогично vmselect):
  affinity: ... 

  # Ресурсы (аналогично vmselect):
  resources: ... 

  # Мониторинг:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8480"

# === vmstorage ===
# Компонент хранения (аналог persistent слоя БД)
vmstorage:
  enabled: true         # Включение компонента
  replicaCount: 3       # 3 реплики для кворума (N/2+1)
  fullnameOverride: vmstorage  # Фиксированное имя ресурсов
  
  # Параметры работы:
  extraArgs:
    retentionPeriod: 30d  # Автоудаление данных старше 30 дней

  # Распределение подов (аналогично vmselect):
  affinity: ...

  # Хранилище:
  persistentVolume:
    enabled: true       # Включить постоянное хранилище
    storageClass: longhorn-db  # Использовать распределенный storageClass
    mountPath: /storage # Путь монтирования
    size: 8Gi           # Размер тома (рекомендуется 50Gi+ для production)

  # Ресурсы:
  resources:
    limits:
      cpu: 100m       # Лимит: 0.1 ядра CPU
      memory: 1024Mi  # Лимит: 1 GiB RAM
    requests:
      cpu: 50m        # Гарантировано: 0.05 ядра CPU
      memory: 500Mi    # Гарантировано: 500 MiB RAM

  # Мониторинг:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8482"
```

### Ключевые особенности:

1. **Архитектура HA**:
   - Все компоненты развернуты в 3 репликах
   - Строгий podAntiAffinity для распределения по разным нодам
   - Коэффициент репликации данных = 3

2. **Хранение данных**:
   - Используется распределенное хранилище (longhorn-db)
   - Автоматическое удаление старых данных через 30 дней
   - Минимальный объем диска (8Gi) - требует увеличения для production

3. **Мониторинг**:
   - Все компоненты экспортируют метрики на стандартных портах
   - Аннотации для автоматического обнаружения Prometheus

4. **Ресурсы**:
   - Указаны минимальные значения (недостаточные для production)
   - Для production рекомендуется:
     - vmselect: 1 CPU / 1Gi RAM
     - vminsert: 2 CPU / 2Gi RAM
     - vmstorage: 2 CPU / 4Gi RAM


### Проверка работы:
```bash
# Проверить распределение подов:
kubectl get pods -l 'app in (vmselect,vminsert,vmstorage)' -o wide

# Проверить логи vmstorage:
kubectl logs -f <vmstorage-pod> | grep -i error

# Проверить метрики:
curl http://<vmselect-service>:8481/metrics | grep -i health
```