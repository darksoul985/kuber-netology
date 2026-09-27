# Домашнее задание к занятию «Как работает сеть в K8s»

Манифесты:

[deployment.yaml](deployment.yaml)

[services.yaml](services.yaml)

[namespace.yaml](namespace.yaml)

[allow-to-dns.yaml](allow-to-dns.yaml)

[backent-to-cache.yaml](backent-to-cache.yaml)

[default-deny.yaml](default-deny.yaml)

[frontend-to-backend.yaml](frontend-to-backend.yaml)

Запущенные манифесты:

![list_po_svc_deployment_np](list_po_svc_deployment_np.png)

## Проверка разрешённого трафика

### Frontend:

![front-allow](front-allow.png)

### Backend:

![back-allow](back-allow.png)

### Cache:

![cache-allow](cache-allow.png)
