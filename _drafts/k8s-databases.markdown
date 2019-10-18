Итак, есть у нас великая мечта избавиться от "Распределенного монолита", когда вроде как монолит попилен на микросервисы, но они по прежнему продолжают ходить в одну общую базу и имеют все возможности орудовать "мимо кассы" и править данные напрямую в базе, минуя барьеры бизнес-логики. Есть только один радикальный способ решить эту проблему -- выделить каждому микросервису свою собственнуюо базу данных, не базу внутри одного инстанса СУБД, а именно отдельный инстанс.

Это создает административный взрыв: разворачивать, монитортить и бекапить даже одну базу -- задача, которой каждый предпочел бы не заниматься, если бы была возможность. Иметь десятки разных баз кажется за гранью разумного. Делать все это по щелчку (новый микросервис/ветка/окружение) -- кажется невозможным.

## Изобретая решение

Если на секунду отвлечься и взглянуть на проблему издалека, то возникает вопрос "Почему stateful-приложения вызывают столько боли?". Рефлекторный ответ будет звучать примерно так: "Ну, у него же есть состояние, мы не можем его просто так убивать, пересоздавать и скейлить, как стейтлесс-приложения". В этом есть доля правды, но такой ответ подразумевает будто бы у нас нет никакой возни со стейтлесс-приложениями и "оно там вообще все само".

А давайте вспомним, благодаря чему "оно само". Если не ходить вокруг да около, то можно сразу указать пальцем на виновника торжества: его величество `Deployment`. Это кубер-контроллер, который берет на себя всю ту самую ненавистную нам возню по манипуляциям реплика-сетами, оставляя на поверхности декларативный интерфейс в виде ямлика. А попробуйте-ка без него реализовать процедуру Rolling Update, выйдет не то, чтобы сильно проще добавления реплики в кластер БД!

А может быть тогда можно и для баз данных и связанной с ними возни сделать такой же контроллер, который предоставит нам такой же YAML-интерфейс? Благо в кубере есть механизм `CustomResourceDefinition`. Он позволяет сделать две вещи:

1) Определить схему кастомного YAML-ресурса, который пользователь может скармливать кластеру
2) Назначить этому кастомному ресурсу оператор -- приложение в кластере, которое отвечает за реализацию хотелок в нем описаных

## KubeDB -- декларативные базы данных в Kubernetes

Ребята из AppsCode воспользовались механизмом CRD и реализовали свои кастомные контроллеры для ресурсов, описывающих инстансы популярных СУБД, механизмы их мониторинга, бекапов и восстановления.

Как это выглядит на практике? Вот например Монга:
```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: sample-mongodb
  namespace: demo
spec:
  version: "4.0.11"
  storageType: Durable
  storage:
    storageClassName: "standard"
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  terminationPolicy: WipeOut
```
Этот ресурс в итоге станет подом, сервисом и вольюмом. Да, это стендалоун Монга, но репликация/шардирование настраивается добавлением пары строчек. Оператор возьмет на себя всю возню по сборке кластера.

А вот расписание бекапов:
```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: mongodb-backup
  namespace: demo
spec:
  schedule: "0 2 * * *"
  task:
    name: mongodb-backup-4.0.11
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: sample-mongodb
  retentionPolicy:
    name: daily-weekly-monthy-for-a-year
    keepDaily: 7
    keepWeekly: 4
    keepMonthly: 12
    prune: true
```
Этот ямлик создаст `CronJob`, который будет в 2 ночи снимать бекапы и заливать их теплое сухое место `gcs-repo`. Также он возьмет на себя прореживание старых бекапов.

Само теплое и сухое место определяется отдельным ямликом:
```yaml
apiVersion: stash.appscode.com/v1alpha1
kind: Repository
metadata:
  name: gcs-repo
  namespace: demo
spec:
  backend:
    gcs:
      bucket: backups
      prefix: /demo/mongodb
    storageSecretName: gcs-secret
```

Восстановиться из бекапов можно вкинув в кластер `RestoreSession`:
```yaml
apiVersion: stash.appscode.com/v1beta1
kind: RestoreSession
metadata:
  name: sample-mongodb
  namespace: desktop-wrappers-build-api
  labels:
    kubedb.com/kind: MongoDB
spec:
  task:
    name: mongodb-restore-4.0.11
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: sample-mongodb
  rules:
  - snapshots: [latest]
```
Это создаст разовый под, который стянет указанный бекап и нальет его в базу.

## Еще больше автоматизации

Несмотря на то, что это огромный прогресс по сравнению с ручным развертыванием баз, заставлять разработчиков рисовать все эти ямлики -- немного лишнее. Здесь явно прослеживается разделение на индивидуальные и общие компоненты:

- Хранилище и расписание будет скорее всего единственным
- Конфигурация базы будет индивидуальной для каждого проекта

Для этого придумана фишечка `BackupBlueprint`:
```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupBlueprint
metadata:
  name: mongodb-backup-blueprint
spec:
  # ============== Blueprint for Repository ==========================
  backend:
    gcs:
      bucket: my-backups
      prefix: stash-backup/${TARGET_NAMESPACE}/${TARGET_APP_RESOURCE}/${TARGET_NAME}
    storageSecretName: gcs-secret
  # ============== Blueprint for BackupConfiguration =================
  task:
    name: mongodb-backup-${TARGET_APP_VERSION}
  schedule: "0 2 * * *"
  retentionPolicy:
    name: daily-weekly-monthy-for-a-year
    keepDaily: 7
    keepWeekly: 4
    keepMonthly: 12
    prune: true
```

Один раз централизованно создав такой блюпринт, админ кластера дает владельцам приложений включить бекапы для своей базы добавлением одной аннотации:
```yaml
metadata:
  annotations:
    stash.appscode.com/backup-blueprint: mongodb-backup-blueprint
```

Оператор создаст по шаблону описание хранилища и расписание бекапов. Расписание, ротация, куда складывать -- все это не заботит владельца сервиса.
