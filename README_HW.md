# 1. Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

Запущен Depluyment с одной репликой:
![1_replicas_deploy](1_replicas_deploy.png)
Запущен Depluyment с двумя репликами:
![2_replicas_deploy](2_replicas_deploy.png)

deployment - [[dpl_ng_multitool.yaml]]

Создан сервис для доступа к репликам приложений:
![service](service.png)

service - [[service.yaml]]

Создан отдельный Pod с приложением curl и проверен доступ к приложениям:
![success_curl](success_curl.png)

checker - [[checker.yaml]]

# Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

Nginx не стартует при отключенном сервисе. Стартует после запуска Service:
![deploy_with_svc](deploy_with_svc.png)

deployment - [[deploy-delay.yaml]]
service - [[svc-delay.yaml]]
