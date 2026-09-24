# Расчёт требований к кластеру Kubernetes

## 1. Сводная таблица нагрузок (requests)

Расчёт веду по **requests** (гарантированные ресурсы), т.к. именно они влияют на планирование подов. Limits ставлю чуть выше (см. раздел 5).

| Компонент | CPU (на копию) | RAM (на копию) | Копий | Итого CPU | Итого RAM |
|---|---|---|---|---|---|
| БД (отказоустойчивая) | 1 | 4 Gi | 3 | 3 | 12 Gi |
| Кеш (отказоустойчивый) | 1 | 4 Gi | 3 | 3 | 12 Gi |
| Фронтенд | 0.2 | 50 Mi | 5 | 1 | 250 Mi |
| Бекенд | 1 | 600 Mi | 10 | 10 | 6 Gi |
| **Итого по подам** | | | | **17 CPU** | **~30.5 Gi** |

> Примечание: 30.5 Gi — это сумма requests. Реальное потребление ОЗУ обычно ниже (особенно у БД/кеша), но планировать нужно от requests.

## 2. Требования к кластеру (worker-ноды)

### CPU
17 CPU — это только сумма requests. Добавляем:
- **System reserved** (kubelet, CRI, ОС): ~10–15% → +2.5 CPU
- **Запас на всплески и неучтённые поды** (ingress, DNS, monitoring, service mesh, операторы): ~20–30%

Итого по CPU: **~24–26 vCPU** минимум для «плотной» упаковки.

### RAM
30.5 Gi requests + system reserved (~2–4 Gi на ноду) + запас. Ориентир: **~40–48 Gi RAM** по кластеру.

### Рекомендуемая конфигурация воркеров

**Вариант «production-grade» (рекомендую): 3 worker-ноды**
- Каждая: **8 vCPU / 16 Gi RAM**
- Итого: 24 vCPU / 48 Gi RAM
- Плюс отказоустойчивость: падение одной ноды не уронит всё.

**Вариант «экономный»: 2 ноды**
- Каждая: **12 vCPU / 24 Gi RAM**
- Итого: 24 vCPU / 48 Gi RAM
- Минус: при падении ноды теряется половина ёмкости, могут не влезть поды БД/кеша.

**Вариант «максимальная отказоустойчивость»: 4 ноды**
- Каждая: **8 vCPU / 16 Gi RAM**
- Итого: 32 vCPU / 64 Gi RAM
- Позволяет пережить падение ноды без деградации.

## 3. Control plane (master-ноды)

Для production — **3 master-ноды** (HA etcd):
- Каждая: **2–4 vCPU / 4–8 Gi RAM / SSD**
- Managed-кластер (EKS/GKE/AKS) — этот слой берёт на себя провайдер.

## 4. Отказоустойчивость (важные нюансы)

### БД и кеш — это StatefulSet, а не Deployment
- **3 копии БД** — это, скорее всего, primary + 2 replica или Patroni/Stolon-кластер. PodAntiAffinity обязателен, чтобы реплики не сели на одну ноду.
- **Кеш** — Redis Cluster / Sentinel / KeyDB. Тоже anti-affinity + topologySpreadConstraints.
- Из-за anti-affinity **нужно минимум 3 worker-ноды** — иначе 3 реплики БД физически не разъедутся.

### PDB (PodDisruptionBudget)
- Для БД и кеша: `minAvailable: 2` из 3 (или `maxUnavailable: 1`).
- Для бекенда: `minAvailable: 7` из 10.
- Для фронта: `minAvailable: 3` из 5.

### Storage для БД
- PersistentVolume: **SSD/NVMe**, минимум **50–100 Gi** на реплику (зависит от данных).
- StorageClass с `WaitForFirstConsumer`.
- Итог: **3 × 100 Gi = 300 Gi** быстрого диска только под БД (+ бэкапы).

### Storage для кеша
- Обычно кеш держат в памяти, но если нужен persistence (RDB/AOF) — **20–50 Gi** на реплику.
- Или вообще без PV, если кеш — чисто in-memory.

## 5. Requests vs Limits (best practice)

| Компонент | CPU req | CPU lim | RAM req | RAM lim |
|---|---|---|---|---|
| БД | 1 | 2 | 4 Gi | 4.5–5 Gi |
| Кеш | 1 | 2 | 4 Gi | 4.5–5 Gi |
| Фронт | 0.1 | 0.5 | 50 Mi | 128 Mi |
| Бек | 0.5 | 2 | 600 Mi | 1 Gi |

- **CPU limit** для БД/кеша/бекенда ставить не всегда хорошо — throttling бьёт по latency. Лучше `requests = limits` для CPU у latency-sensitive сервисов, либо вообще без CPU limit.
- **RAM limit** обязательно = близко к requests для stateful-сервисов, иначе OOMKilled.

## 6. Ingress / Service / HPA

- **Ingress Controller** (nginx/traefik): 2–3 реплики, по 0.5 CPU / 512 Mi → +1.5 CPU / 1.5 Gi.
- **CoreDNS**: 2 реплики, по 0.1 CPU / 128 Mi.
- **HPA** для фронта и бекенда:
  - Фронт: min 5, max 15, target CPU 60%.
  - Бек: min 10, max 40, target CPU 70%.
  - БД и кеш — **не HPA**, а ручное/операторное масштабирование.

## 7. Итоговый чек-лист ресурсов кластера

```
Worker-ноды:           3 × (8 vCPU / 16 Gi RAM)   = 24 vCPU / 48 Gi
Master-ноды (HA):      3 × (4 vCPU / 8 Gi RAM)    = 12 vCPU / 24 Gi  (или managed)
Storage (БД):          3 × 100 Gi NVMe SSD        = 300 Gi
Storage (кеш):         3 × 30 Gi  SSD             = 90 Gi (опционально)
Storage (бэкапы):      S3 / NFS                   = 200+ Gi
Ingress + CoreDNS:     ~2 vCPU / 2 Gi
```

## 8. Про Helm-чарт (упаковка под окружения)

Структура чарта:
```
myapp/
├── Chart.yaml
├── values.yaml              # дефолты
├── values-dev.yaml          # 1 реплика БД/кеша (без HA), 1 фронт, 2 бек
├── values-staging.yaml      # 3 БД, 3 кеш, 3 фронт, 5 бек
├── values-prod.yaml         # 3 БД, 3 кеш, 5 фронт, 10 бек, anti-affinity, PDB
└── templates/
    ├── database-statefulset.yaml
    ├── cache-statefulset.yaml
    ├── backend-deployment.yaml
    ├── frontend-deployment.yaml
    ├── ingress.yaml
    ├── pdb.yaml
    ├── hpa.yaml
    └── servicemonitor.yaml
```

Ключевые best practices для чарта:
- **Реплики, ресурсы, storage-классы — через values.yaml**, не хардкодить.
- **PodAntiAffinity** для БД/кеша включать только в prod (в dev — не нужно).
- **PDB** — только в prod.
- **Secrets** — через External Secrets Operator / Sealed Secrets, не в values.
- **`helm lint` + `helm template` + kubeconform** в CI.
- **Отдельные namespace** под каждое окружение.
- **ArgoCD/Flux** для GitOps-деплоя.

## 9. Итого

Минимально жизнеспособный **prod-кластер**:
- **3 worker-ноды по 8 vCPU / 16 Gi RAM** (24 vCPU / 48 Gi суммарно),
- **3 master-ноды** (или managed control plane),
- **~400 Gi SSD** под stateful-нагрузки,
- Ingress + CoreDNS + мониторинг (~2 vCPU / 2 Gi сверху),
- Helm-чарт с values-файлами на dev/staging/prod,
- StatefulSet + anti-affinity + PDB для БД и кеша, Deployment + HPA для фронта и бекенда.

Если бюджет ограничен — можно ужаться до **2 нод по 12 vCPU / 24 Gi**, но тогда теряется отказоустойчивость на уровне нод (anti-affinity для 3 реплик БД не сработает).
