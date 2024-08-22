# Дипломный практикум в Yandex.Cloud
  * [Цели:](#цели)
  * [Этапы выполнения:](#этапы-выполнения)
     * [Создание облачной инфраструктуры](#создание-облачной-инфраструктуры)
     * [Создание Kubernetes кластера](#создание-kubernetes-кластера)
     * [Создание тестового приложения](#создание-тестового-приложения)
     * [Подготовка cистемы мониторинга и деплой приложения](#подготовка-cистемы-мониторинга-и-деплой-приложения)
     * [Установка и настройка CI/CD](#установка-и-настройка-cicd)
  * [Что необходимо для сдачи задания?](#что-необходимо-для-сдачи-задания)
  * [Как правильно задавать вопросы дипломному руководителю?](#как-правильно-задавать-вопросы-дипломному-руководителю)

**Перед началом работы над дипломным заданием изучите [Инструкция по экономии облачных ресурсов](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD).**


---
## Цели:

1. Подготовить облачную инфраструктуру на базе облачного провайдера Яндекс.Облако.
2. Запустить и сконфигурировать Kubernetes кластер.
3. Установить и настроить систему мониторинга.
4. Настроить и автоматизировать сборку тестового приложения с использованием Docker-контейнеров.
5. Настроить CI для автоматической сборки и тестирования.
6. Настроить CD для автоматического развёртывания приложения.

---
## Этапы выполнения:


### Создание облачной инфраструктуры

Для начала необходимо подготовить облачную инфраструктуру в ЯО при помощи [Terraform](https://www.terraform.io/).

Особенности выполнения:

- Бюджет купона ограничен, что следует иметь в виду при проектировании инфраструктуры и использовании ресурсов;
Для облачного k8s используйте региональный мастер(неотказоустойчивый). Для self-hosted k8s минимизируйте ресурсы ВМ и долю ЦПУ. В обоих вариантах используйте прерываемые ВМ для worker nodes.

Предварительная подготовка к установке и запуску Kubernetes кластера.

1. Создайте сервисный аккаунт, который будет в дальнейшем использоваться Terraform для работы с инфраструктурой с необходимыми и достаточными правами. Не стоит использовать права суперпользователя
2. Подготовьте [backend](https://www.terraform.io/docs/language/settings/backends/index.html) для Terraform:  
   а. Рекомендуемый вариант: S3 bucket в созданном ЯО аккаунте(создание бакета через TF)
   б. Альтернативный вариант:  [Terraform Cloud](https://app.terraform.io/)  

[Создание бакета и регистри](https://github.com/Dmitrywh1/infrastructure/tree/main/bucket)

3. Создайте VPC с подсетями в разных зонах доступности.

[Создание VPC](https://github.com/Dmitrywh1/infrastructure/tree/main/compute_instance)

4. Убедитесь, что теперь вы можете выполнить команды `terraform destroy` и `terraform apply` без дополнительных ручных действий.

*Реализовано в GitLab CI*



5. В случае использования [Terraform Cloud](https://app.terraform.io/) в качестве [backend](https://www.terraform.io/docs/language/settings/backends/index.html) убедитесь, что применение изменений успешно проходит, используя web-интерфейс Terraform cloud.

Ожидаемые результаты:

1. Terraform сконфигурирован и создание инфраструктуры посредством Terraform возможно без дополнительных ручных действий.

2. Полученная конфигурация инфраструктуры является предварительной, поэтому в ходе дальнейшего выполнения задания возможны изменения.
![image](https://github.com/user-attachments/assets/ed4baf43-3217-411b-944c-423273afcdbe)


---
### Создание Kubernetes кластера

На этом этапе необходимо создать [Kubernetes](https://kubernetes.io/ru/docs/concepts/overview/what-is-kubernetes/) кластер на базе предварительно созданной инфраструктуры.   Требуется обеспечить доступ к ресурсам из Интернета.

Это можно сделать двумя способами:

1. Рекомендуемый вариант: самостоятельная установка Kubernetes кластера.  
   а. При помощи Terraform подготовить как минимум 3 виртуальных машины Compute Cloud для создания Kubernetes-кластера. Тип виртуальной машины следует выбрать самостоятельно с учётом требовании к производительности и стоимости. Если в дальнейшем поймете, что необходимо сменить тип инстанса, используйте Terraform для внесения изменений.  
   б. Подготовить [ansible](https://www.ansible.com/) конфигурации, можно воспользоваться, например [Kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/)  
   в. Задеплоить Kubernetes на подготовленные ранее инстансы, в случае нехватки каких-либо ресурсов вы всегда можете создать их при помощи Terraform.

2. Альтернативный вариант: воспользуйтесь сервисом [Yandex Managed Service for Kubernetes](https://cloud.yandex.ru/services/managed-kubernetes)  
  а. С помощью terraform resource для [kubernetes](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_cluster) создать **региональный** мастер kubernetes с размещением нод в разных 3 подсетях      
  б. С помощью terraform resource для [kubernetes node group](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_node_group)
  
Ожидаемый результат:

1. Работоспособный Kubernetes кластер.
[Манифесты k8s](https://github.com/Dmitrywh1/infrastructure/tree/main/kubernetes)

```
Манифесты, отличающиеся от нативного kubespray:
 - ../k8s-cluster/addons.yml: добавлена установка helm и ingress-nginx
 - ../group_vars/kube_ingress.yml: taints и tolerations для ingress-нод
 - ../manifests/*: манифесты для конфигурирования NS и создания Ingress для Grafana
```

2. В файле `~/.kube/config` находятся данные для доступа к кластеру.
[scrape kubeconfig](https://github.com/Dmitrywh1/infrastructure/blob/main/kubernetes/.install-kubespray.gitlab-ci.yml)

```
kubeconfig вытягивается отдельной ci-джобой и сохраняется в артифактах для дальнейшего переиспользования
```

3. Команда `kubectl get pods --all-namespaces` отрабатывает без ошибок.

![image](https://github.com/user-attachments/assets/09d126ff-cdd4-484f-9632-1ed9ee9ed61c)

Описание созданного кластера:
[Конфигурировать количество нод каждой группы можно в ci джобе](https://github.com/Dmitrywh1/infrastructure/blob/main/compute_instance/.compute_instance.gitlab-ci.yml)

![image](https://github.com/user-attachments/assets/6ba0e498-18ed-4c1b-b65f-a4e651dbdc1f)


```
Сетевая доступность к кластеру реализована при помощи LB(L4), запросы принимает Ingress-worker node, далее роутится на воркер ноды.
Сетевая доступность из кластера реализована при помощи nat_gateway от YC

```

![image](https://github.com/user-attachments/assets/36e24a34-8d6e-45ab-9db8-5c486fe87fff)


---
### Создание тестового приложения

Для перехода к следующему этапу необходимо подготовить тестовое приложение, эмулирующее основное приложение разрабатываемое вашей компанией.

Способ подготовки:

1. Рекомендуемый вариант:  
   а. Создайте отдельный git репозиторий с простым nginx конфигом, который будет отдавать статические данные.  
   б. Подготовьте Dockerfile для создания образа приложения.  
2. Альтернативный вариант:  
   а. Используйте любой другой код, главное, чтобы был самостоятельно создан Dockerfile.

Ожидаемый результат:

1. Git репозиторий с тестовым приложением и Dockerfile.

[samplefe](https://github.com/Dmitrywh1/samplefe/tree/c22907c72ee4c0f0c5036eb0510468ebbfba8c94)

2. Регистри с собранным docker image. В качестве регистри может быть DockerHub или [Yandex Container Registry](https://cloud.yandex.ru/services/container-registry), созданный также с помощью terraform.
![image](https://github.com/user-attachments/assets/f1338874-b9cf-40f0-8b10-21f5846d3160)

---
### Подготовка cистемы мониторинга и деплой приложения

Уже должны быть готовы конфигурации для автоматического создания облачной инфраструктуры и поднятия Kubernetes кластера.  
Теперь необходимо подготовить конфигурационные файлы для настройки нашего Kubernetes кластера.

Цель:
1. Задеплоить в кластер [prometheus](https://prometheus.io/), [grafana](https://grafana.com/), [alertmanager](https://github.com/prometheus/alertmanager), [экспортер](https://github.com/prometheus/node_exporter) основных метрик Kubernetes.
2. Задеплоить тестовое приложение, например, [nginx](https://www.nginx.com/) сервер отдающий статическую страницу.

Способ выполнения:
1. Воспользоваться пакетом [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus), который уже включает в себя [Kubernetes оператор](https://operatorhub.io/) для [grafana](https://grafana.com/), [prometheus](https://prometheus.io/), [alertmanager](https://github.com/prometheus/alertmanager) и [node_exporter](https://github.com/prometheus/node_exporter). Альтернативный вариант - использовать набор helm чартов от [bitnami](https://github.com/bitnami/charts/tree/main/bitnami).

2. Если на первом этапе вы не воспользовались [Terraform Cloud](https://app.terraform.io/), то задеплойте и настройте в кластере [atlantis](https://www.runatlantis.io/) для отслеживания изменений инфраструктуры. Альтернативный вариант 3 задания: вместо Terraform Cloud или atlantis настройте на автоматический запуск и применение конфигурации terraform из вашего git-репозитория в выбранной вами CI-CD системе при любом комите в main ветку. Предоставьте скриншоты работы пайплайна из CI/CD системы.

![image](https://github.com/user-attachments/assets/7fd00d02-ab1d-46d1-8d89-49f13570ff3c)


Ожидаемый результат:
1. Git репозиторий с конфигурационными файлами для настройки Kubernetes.

 [Установка мониторинга](https://github.com/Dmitrywh1/infrastructure/blob/main/kubernetes/.prepare-k8s-.gitlab-ci.yml)

![image](https://github.com/user-attachments/assets/1fa47f9e-1b56-426a-88e8-10a63dd5502a)


3. Http доступ к web интерфейсу grafana.

![image](https://github.com/user-attachments/assets/33d1a45f-c986-4f31-908a-571ed105c951)


3. Дашборды в grafana отображающие состояние Kubernetes кластера.

![image](https://github.com/user-attachments/assets/a33bc527-403e-45ff-8e03-cd5a352e3ad3)


4. Http доступ к тестовому приложению.

![image](https://github.com/user-attachments/assets/be3451d5-7cf8-4fae-a089-eb3e09ba5639)


---
### Установка и настройка CI/CD

Осталось настроить ci/cd систему для автоматической сборки docker image и деплоя приложения при изменении кода.

Цель:

1. Автоматическая сборка docker образа при коммите в репозиторий с тестовым приложением.
2. Автоматический деплой нового docker образа.

Можно использовать [teamcity](https://www.jetbrains.com/ru-ru/teamcity/), [jenkins](https://www.jenkins.io/), [GitLab CI](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) или GitHub Actions.

Ожидаемый результат:

1. Интерфейс ci/cd сервиса доступен по http.

![image](https://github.com/user-attachments/assets/76db556e-3dc4-4c84-8248-d79f59d92e0c)


2. При любом коммите в репозиторие с тестовым приложением происходит сборка и отправка в регистр Docker образа.

![image](https://github.com/user-attachments/assets/c1e2088b-77b8-4f3f-b73c-c3c3ed6b8bb6)


3. При создании тега (например, v1.0.0) происходит сборка и отправка с соответствующим label в регистри, а также деплой соответствующего Docker образа в кластер Kubernetes.

   ![image](https://github.com/user-attachments/assets/1f6e9ae1-db4e-4853-ac63-19921c0bc566)

[.gitlab-ci.yml](https://github.com/Dmitrywh1/samplefe/blob/c22907c72ee4c0f0c5036eb0510468ebbfba8c94/.gitlab-ci.yml)

---
## Что необходимо для сдачи задания?

1. Репозиторий с конфигурационными файлами Terraform и готовность продемонстрировать создание всех ресурсов с нуля.

[infrastructure](https://github.com/Dmitrywh1/infrastructure/tree/main)
2. Пример pull request с комментариями созданными atlantis'ом или снимки экрана из Terraform Cloud или вашего CI-CD-terraform pipeline.

![image](https://github.com/user-attachments/assets/df60132b-b7ae-43d8-bdab-5559b39ac691)

3. Репозиторий с конфигурацией ansible, если был выбран способ создания Kubernetes кластера при помощи ansible.

[Установка через ci джобу](https://github.com/Dmitrywh1/infrastructure/blob/main/kubernetes/.install-kubespray.gitlab-ci.yml)
4. Репозиторий с Dockerfile тестового приложения и ссылка на собранный docker image.

[samplefe](https://github.com/Dmitrywh1/samplefe)
5. Репозиторий с конфигурацией Kubernetes кластера.

[k8s](https://github.com/Dmitrywh1/infrastructure/tree/main/kubernetes)
6. Ссылка на тестовое приложение и веб интерфейс Grafana с данными доступа.
7. Все репозитории рекомендуется хранить на одном ресурсе (github, gitlab)

