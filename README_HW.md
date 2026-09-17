# Задание 1: Настройка Service (ClusterIP и NodePort)

Доступ к сервису с ClusterIP:
![clusterip_curl](clusterip_curl.png)

Доступ к сервису с NodePort хоста:
![nodeport_curl](nodeport_curl.png)

Манифесты:
[deployment-multi-container](deployment-multi-container.yaml)
[service-clusterip](service-clusterip.yaml)
[service-nodeport](service-nodeport.yaml)

# Задание 2: Настройка Ingress

Curl запросы к сервисам:
![curl_back_front](ingress_curl.png)

Насколько я понял, в случае с запросом /api, так как wbitt/network-multitool nginx внутри не является полноценным маршрутизатором, из за чего и возвращается 404 от nginx внутри контейнера.

Манифесты:
[deployment-frontend](deployment-frontend.yaml)
[deployment-backend](deployment-backend.yaml)
[service-frontend](service-frontend.yaml)
[service-backend](service-backend.yaml)
[ingress](ingress.yaml)
