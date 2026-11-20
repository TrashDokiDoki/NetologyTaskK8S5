# Домашнее задание к занятию «Хранение в K8s»

## Задание 1. Volume: обмен данными между контейнерами в Pod

### 1. Создать Deployment из контейнеров busybox и multitool

Создан Deployment `data-exchange`, состоящий из двух контейнеров:

- `busybox`;
- `multitool`.

Оба контейнера используют общий том `emptyDir`, смонтированный в директорию `/shared`.

[Манифест containers-data-exchange.yaml](containers-data-exchange.yaml)

---

### 2. Настроить busybox на запись данных каждые 5 секунд

Контейнер `busybox` выполняет цикл:

```
while true; do
  echo "$(date)" >> /shared/data.txt
  sleep 5
done
```

Таким образом, каждые 5 секунд в файл `/shared/data.txt` записывается текущая дата и время.

Проверка Pod:

```
kubectl describe pods -l app=data-exchange
```

В описании видно, что оба контейнера используют общий том:

```
Volumes:
  shared-data:
    Type: EmptyDir
```

<img width="932" height="939" alt="image" src="https://github.com/user-attachments/assets/8ce9dad3-4b8a-49e5-90d7-17ddf4fce2d4" />
<img width="1405" height="408" alt="image" src="https://github.com/user-attachments/assets/3453ba71-554b-4b0a-b1df-e4208de28e1d" />

---

### 3. Обеспечить чтение файла контейнером multitool

Контейнер `multitool` использует тот же том `/shared`.

Для проверки выполнена команда:

```
kubectl exec -it deployment/data-exchange -c multitool -- tail -f /shared/data.txt
```

Получен результат:

```
Fri Nov 20 14:52:49 UTC 2026
Fri Nov 20 14:52:54 UTC 2026
Fri Nov 20 14:52:59 UTC 2026
```

Новые строки появляются каждые 5 секунд.

Это подтверждает, что контейнер `busybox` записывает данные в общий том, а контейнер `multitool` может читать эти данные.

<img width="818" height="77" alt="image" src="https://github.com/user-attachments/assets/316393a5-868e-4ba6-b91c-62f223e5e0c4" />

**Результат:** обмен данными между двумя контейнерами внутри одного Pod успешно реализован с использованием `emptyDir`.

---

# Задание 2. PersistentVolume и PersistentVolumeClaim

## 1. Создать Deployment, использующий PVC

Создан Deployment `data-exchange-pvc`, состоящий из двух контейнеров:

- `busybox`;
- `multitool`.

Вместо временного `emptyDir` контейнеры используют PersistentVolumeClaim `local-pvc`.

[Манифест pv-pvc.yaml](pv-pvc.yaml)

---

## 2. Создать PV и PVC для локальной директории ноды

Создан PersistentVolume:

```
local-pv
```

Объём:

```
1Gi
```

Режим доступа:

```
ReadWriteOnce
```

Политика освобождения:

```
Retain
```

Локальная директория ноды:

```
/tmp/netology-pv
```

Также создан PVC `local-pvc`.

Из-за наличия стандартного StorageClass в MicroK8s для PVC было явно установлено:

```
storageClassName: ""
```

Это позволило привязать PVC к созданному вручную PV.

Проверка:

```
kubectl get pv
kubectl get pvc
kubectl get pods
```

Результат:

```
local-pv    1Gi   RWO   Retain   Bound   default/local-pvc
```

```
local-pvc   Bound   local-pv   1Gi   RWO
```

Pod приложения:

```
data-exchange-pvc-...   2/2   Running
```

<img width="978" height="302" alt="image" src="https://github.com/user-attachments/assets/8ad478b7-1add-46dd-9a57-27590d1fff67" />

---

## 3. Проверить чтение данных из PersistentVolume

В контейнере `multitool` выполнена команда:

```
kubectl exec -it deployment/data-exchange-pvc -c multitool -- tail -f /shared/data.txt
```

Результат:

```
Fri Nov 20 14:56:55 UTC 2026
Fri Nov 20 14:57:00 UTC 2026
Fri Nov 20 14:57:05 UTC 2026
Fri Nov 20 14:57:10 UTC 2026
```

Данные успешно записываются контейнером `busybox` и читаются контейнером `multitool`.

Дополнительно файл был проверен непосредственно на ноде:

```
sudo ls -la /tmp/netology-pv
sudo tail /tmp/netology-pv/data.txt
```

Файл `data.txt` присутствует в локальной директории ноды.

<img width="483" height="245" alt="image" src="https://github.com/user-attachments/assets/5e12595e-862f-4517-819b-c6ce4140df61" />

---

## 4. Удалить Deployment и PVC и проверить состояние PV

Deployment и PVC были удалены:

```
kubectl delete deployment data-exchange-pvc
kubectl delete pvc local-pvc
```

После удаления PVC PersistentVolume перестаёт быть связан с существующим PVC.

Для проверки состояния PV используется:

```
kubectl describe pv local-pv
```

При политике:

```
persistentVolumeReclaimPolicy: Retain
```

PV не удаляет связанные с ним физические данные автоматически.

<img width="1032" height="404" alt="image" src="https://github.com/user-attachments/assets/ad3d6150-0f18-4680-9529-8eae074277e2" />

### Объяснение

Политика `Retain` означает, что после удаления PVC Kubernetes сохраняет PersistentVolume и данные, находящиеся в физическом хранилище.

Такое поведение защищает данные от автоматического удаления вместе с PVC.

---

## 5. Проверить файл после удаления PVC и PV

После удаления Deployment и PVC файл всё ещё присутствовал на локальной ноде:

```
sudo ls -la /tmp/netology-pv
sudo tail /tmp/netology-pv/data.txt
```

Результат показал, что `data.txt` сохранился.

После этого PersistentVolume был удалён:

```
kubectl delete pv local-pv
```

Повторная проверка:

```
sudo ls -la /tmp/netology-pv
sudo tail /tmp/netology-pv/data.txt
```

показала, что файл также остался:

```
/tmp/netology-pv/data.txt
```

<img width="593" height="569" alt="image" src="https://github.com/user-attachments/assets/02b64e1d-d1b1-481d-97f1-6b91eeb47374" />

### Почему файл сохранился

В данном задании используется `hostPath`.

PersistentVolume является объектом Kubernetes, который описывает использование директории на ноде:

```
/tmp/netology-pv
```

Удаление объекта PV из Kubernetes не означает удаление самой директории и файлов операционной системы.

Дополнительно для PV была настроена политика:

```
Retain
```

Поэтому Kubernetes не выполняет автоматическое удаление данных.

**Результат:** данные успешно пережили удаление Deployment, PVC и самого объекта PV.

---

# Задание 3. StorageClass

## 1. Создать Deployment с PVC на основе StorageClass

Создан Deployment `data-exchange-sc` из контейнеров:

- `busybox`;
- `multitool`.

Оба контейнера используют PVC `local-sc-pvc`.

[Манифест sc.yaml](sc.yaml)

---

## 2. Создать StorageClass и PVC

Создан StorageClass:

```
local-storage
```

с provisioner:

```
kubernetes.io/no-provisioner
```

и режимом:

```
WaitForFirstConsumer
```

Так как `kubernetes.io/no-provisioner` не создаёт PersistentVolume автоматически, для выполнения задания также был создан локальный PV `local-sc-pv`, относящийся к StorageClass `local-storage`.

Проверка:

```
kubectl get sc
kubectl get pv
kubectl get pvc
kubectl get pods
```

StorageClass:

```
local-storage   kubernetes.io/no-provisioner   WaitForFirstConsumer
```

PV:

```
local-sc-pv   1Gi   RWO   Retain   Bound   default/local-sc-pvc   local-storage
```

PVC:

```
local-sc-pvc   Bound   local-sc-pv   1Gi   RWO   local-storage
```

Pod:

```
data-exchange-sc-...   2/2   Running
```

<img width="1017" height="367" alt="image" src="https://github.com/user-attachments/assets/48688eb5-39cf-4152-89e4-a3eca19df41f" />

---

## 3. Проверить обмен данными

Для чтения файла из контейнера `multitool` выполнена команда:

```
kubectl exec -it deployment/data-exchange-sc -c multitool -- tail -f /shared/data.txt
```

Результат:

```
Fri Nov 20 14:59:55 UTC 2026
Fri Nov 20 15:00:00 UTC 2026
Fri Nov 20 15:00:05 UTC 2026
Fri Nov 20 15:00:10 UTC 2026
Fri Nov 20 15:00:15 UTC 2026
```

Новые строки появляются каждые 5 секунд.

Это подтверждает, что PVC, связанный со StorageClass `local-storage`, успешно используется приложением для хранения и обмена данными.

<img width="836" height="105" alt="image" src="https://github.com/user-attachments/assets/68194f01-455c-4303-adef-c5f4b98528d3" />
