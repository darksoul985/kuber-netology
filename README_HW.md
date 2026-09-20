# Задание 1. Volume: обмен данными между контейнерами в поде

Манифест:

![containers-data-exchange](containers-data-exchange.yaml)

Скриншоты:

![describe_pods1](describe_pods.png)
![describe_pods2](describe_pods1.png)

![logs_output](logs_output.png)

# Задание 2. PV, PVC

PV и PVC подключеные к папке на локальной ноде:

![pv_pvc](pv_pvc.png)

Контейнер multitool может читать данные из файла в смонтированной директории:

![read_from_pv](read_from_pv.png)

Удалены Deployment и PVC:

![del_deploy_pvc](del_deploy_pvc.png)

Объяснение: PV — это ресурс кластера, представляющий собой фрагмент реального хранилища. PV существует независимо от Pod.
После удаления PVC, PV приобретает `Status: Released`, т.е. сообщает что с данным PV нет связанных PVC.
При подключении же статус меняется на `Bound`:

![status_bound](status_bound.png)

Даже после удаления PV данные сохраняются на диске, если параметр `persistentVolumeReclaimPolicy` установлен в значении `Retain`:

![del_pv](del_pv.png)

Манифесты:

[pv-pvc.yaml](pv-pvc.yaml)

# Задание 3. StorageClass

Задача

Создан Deployment приложения, использующего PVC, созданный на основе StorageClass.

![sc_pvc_read_file](sc_pvc_read_file.png)

Манифесты:

[sc.yaml](sc.yaml)
