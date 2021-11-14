# test_deployment
## Задачи и их решения.
- [x] Задача #1
> — у нас мультизональный кластер (три зоны), в котором пять нод  
> — хотим максимально отказоустойчивый deployment
##### Решение
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-app
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - my-app
        topologyKey: "kubernetes.io/hostname"
```
1. **spec.template.spec.topologySpreadConstraints** — решает проблему мультизонального кластера. Дает шедулится только заданному количеству подов на нодах имеющих лэйбл **"topology.kubernetes.io/zone=.*"**. В идеале данный метод стремится распределить все экземпляры подов в разные зоны, но если все зоны уже имеют по экземпляру, на помощь приходит ключ-значение
maxSkew: 1 которое говорит, что максимально допустимая погрешность в количестве подов на зону равно единице. В итоге имея 4 пода и 3 зоны, мы всегда будем иметь два экземпляра в одной из зон.
2. **spec.template.spec.affinity.podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution** — Экземплары подов могут находиться друг с другом на одной ноде только в том случае, если на каждой ноде уже есть по одному экземляру пода.
Результат: kube-scheduler задеплоит 4 экземляра приложения в 3 зоны, в двух зонах будет по одному экземпляру, в третьей два. У нас в кластере есть зона с одной нодой, а правило podAntiAffinity говорит шедулеру не подселять последний экземпляр на ноду которая уже имеет экземпляр, поэтому выбор будет между свободными нодами в зонах с двумя нодами.
- [x] Задача #2

> — приложение требует около 5-10 секунд для инициализации
##### Решение
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 3
  failureThreshold: 3
  successThreshold: 1
livenessProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 2
  successThreshold: 1
```
Чтобы под не падал с ошибкой из-за livenessProbe, нужно добавить readinessProbe. В startupProbe нужды тут не вижу, приложение запускается в временных рамках readinessProbe.  

- [x] Задача #3
> — на первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда “ровно” в районе 128M memory
##### Решение
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 128Mi
```
Добавляем в deployment параметры spec.template.spec.containers.resources. В лимитах нет cpu потому что параметр верхней планки потребления не задан. qosClass у приложения будет Burstable.

- [x] Задача #4
> — по результатам нагрузочного теста известно, что 4 пода справляются с пиковой нагрузкой  
> — приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём  
> — хотим минимального потребления ресурсов от этого deployment’а  
##### Решение
```yaml
replicas: 2
```
Я выставил минимальное количество экземпляров приложения со значением 2, потому что меньшее значение может обернуться даунтаймом при каком-нибудь инцеденте. Для критически важных приложений необходимо иметь от трех реплик.  
```yaml
  minAvailable: 2
```
Желательно выставить PDB значение minAvailable таким же как и минимальное значение replicas. Делается это чтобы при evicting процедуре фактическое количество экземпляров не стало ниже желаемого количества.  
```yaml
  minReplicas: 2
  maxReplicas: 4
  targetCPUUtilizationPercentage: 60
```
Нужно использовать HorizontalPodAutoscaler, которой умеет масштабировать приложение. В данном примере, при достижении 60% использования CPU выставленных в реквестах, HorizontalPodAutoscaler увеличивает количество желаемых реплик в replicasets, в момент когда нагрузка падает — уменьшает.  
Есть возможность использоваться вместо мастабирования по ресурам time-based scaling, но я считаю, что это не лучший вариант, так как бывают сезонные нагрузки и другие глобальные события которые могут повышать и понижать нагрузку на сервис.  
Note: В кластере должен быть установлен metrics-server.  
