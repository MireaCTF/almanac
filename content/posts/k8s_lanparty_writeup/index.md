---
title: "k8s LAN Party write up"
description: ""
date: 2024-06-20
summary: "Обзор тасков с небольшой, но очень интересной стфки по кубу"  

tags: ["pentest"]
showAuthor: true
authors:
  - "cotsom"
---

![Alt text](img/image.png)
Кто в контейнере - привет, остальным соболезную.
Сегодня посмотрим на k8s LAN Party - небольшая CTF от Wiz Research, посвященная Kubernetes.

Платформа представляет собой набор тасков, которые помогут нам разобраться в некоторых базовых концепциях куба, а также познакомят нас с полезными интересными классными механизмами:

- Recon (DNS scanning (https://thegreycorner.com/2023/12/13/kubernetes-internal-service-discovery.html#kubernetes-dns-to-the-partial-rescue))
- Finding Neighbours (sidecars (https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/))
- Data Leakage
- Bypassing Boundaries
- Lateral Movement (administrative services (https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#request))


Консоль для взаимодействия со средой находится в самом вебе, что очень удобно.

## DNSing with the stars
**Описание:** 
*You have shell access to compromised a Kubernetes pod at the bottom of this page, and your* *next objective is to compromise other internal services further.*
*As a warmup, utilize [DNS scanning](https://thegreycorner.com/2023/12/13/kubernetes-internal-service-discovery.html#kubernetes-dns-to-the-partial-rescue) to uncover hidden internal services and obtain the flag. We have "loaded your machine with dnscan to ease this process for further challenges.*


Итак мы находимся внутри пода и перед нами стоит задача найти другие скрытые сервисы.

В kubernetes имеется [внутренний DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/), который поможет нам в Service Discovery.

В описании уже сказано, что для этого мы можем использовать утилиту [dnscan](https://gist.github.com/nirohfeld/c596898673ead369cb8992d97a1c764e), которую разработчики заботливо положили в наш контейнер. Данная утилита пройдется по всей подсети и запросит у внутреннего DNS PTR записи, выполняя reverse lookup.

Но как нам узнать внутреннуюю подсеть?

1. Мы можем узнать адрес KubeAPI, к которому идут все запросы, через env
![Alt text](image.png)
2. Заглянуть в /etc/resolv.conf
![Alt text](image-1.png)

Далее запустим dnscan для обнаружения сервиса 

`dnscan -subnet 10.100.0.0/16`

курлим найденный сервис и забираем флаг.
![Alt text](image-2.png)


## Hello?
**Описание:**
Sometimes, it seems we are the only ones around, but we should always be on guard against invisible [sidecars](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) reporting sensitive secrets.

Из описания понятно, что где то рядом с нами притаился sidecar контейнер.

Если в кратце, то сайдкар контейнеры нужны для того, чтобы мы могли расширить функцонал нашего приложения, не внося в него изменений. Они находятся в одном поде с основным сервисом и общаются с ним посредством заложенного разработчиком механизма (вольюмы, http запросы etc.). Посмотреть пример можно [тут](https://habr.com/ru/companies/nixys/articles/559368/).

Наша задача услышать, что же наш сосед пытается нам донести. 

Для начала проведем днс сканирование с помощью уже знакомого нам dnscan.

`dnscan -subnet 10.100.0.0/16`

Мы обнаружили некий `reporting service`. Вероятно это и есть наш сайдкар.
![Alt text](image-3.png)

Если мы попытаемся просто курлануть его, то очевидно ничего не выйдет.

Давайте просто попробуем прослушать весь сетевой трафик.

Замечаем, что в контейнере уже заранее лежит tcpdump и понимаем, что мы на верном пути с:
Запустим его с флагом `-A` для чтения пакетов.

`tcpdump -A`

И мы успешно замечаем флаг в http пакете, который нам отправил наш соседний контейнер.

![Alt text](image-4.png)


## Exposed File Share
**Описание:**
*The targeted big corp utilizes outdated, yet cloud-supported technology for data storage in production. But oh my, this technology was introduced in an era when access control was only network-based 🤦‍️.*

Видим file share - смотрим, что примонтировано в наш контейнер.
![Alt text](image-5.png)

Видим смонтированную amazon EFS шару по пути `/efs`. Находим в ней флаг, который не можем прочитать.
![Alt text](image-6.png)

Просмотрев первую подсказку мы узнаем про некие инструменты nfs-ls, nfs-cat и находим [документацию](https://github.com/sahlberg/libnfs).
![Alt text](image-7.png)

Пробуем составить команду, с помощью которой мы сможем взаимодействовать с хранилищем.

```
player@wiz-k8s-lan-party:~$ nfs-ls "nfs://fs-0779524599b7d5e7e.efs.us-west-1.amazonaws.com//?version=4"

----------  1     1     1           73 flag.txt
```

Отлично, мы смогли вывести его содержимое. 

Однако, если мы попытаемся прочитать флаг, то получим ошибку.

```
player@wiz-k8s-lan-party:~$ nfs-cat "nfs://fs-0779524599b7d5e7e.efs.us-west-1.amazonaws.com//?version=4"

Failed to open file /: open call failed with "NFS4: (path /) failed with NFS4ERR_INVAL(-22)"
Failed to open nfs://fs-0779524599b7d5e7e.efs.us-west-1.amazonaws.com//?version=4
```

Нагуглив [доку от aws](https://docs.aws.amazon.com/en_en/efs/latest/ug/accessing-fs-nfs-permissions.html) можно понять, что нам не хватает UID/GID

*After creating a file system, by default only the root user (UID 0) has read, write, and execute permissions.*

Следуя все тому же ману из подсказки, составляем новую нагрузку и получаем флаг

`nfs-cat "nfs://fs-0779524599b7d5e7e.efs.us-west-1.amazonaws.com//flag.txt?version=4&uid=0"`


## The Beauty and The Ist
**Описание:**
*Apparently, new service mesh technologies hold unique appeal for ultra-elite users (root users). Don't abuse this power; use it responsibly and with caution.*

**Policy:**
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: istio-get-flag
  namespace: k8s-lan-party
spec:
  action: DENY
  selector:
    matchLabels:
      app: "{flag-pod-name}"
  rules:
  - from:
    - source:
        namespaces: ["k8s-lan-party"]
    to:
    - operation:
        methods: ["POST", "GET"]
```

Как сказано на официальном ресурсе, istio расширяет дефолтные возможности кубера, позволяя более гибко настроить возможности для сетевой инфраструктуры приложений, таких как балансировка нагрузки, обнаружение сервисов, **контроль доступа**, шифрование, мониторинг и трассировка.

Более подробно про этот Service Mesh можно прочитать в [этой статье](https://habr.com/ru/companies/oleg-bunin/articles/726958/)

Нам же исходя из определения и представленной политики достаточно понимать, что istio ограничивает нам доступ из нашего неймспейса до нужного сервиса, в рамках этого задания.

Просканируем сеть старым способом, найдем этот сервис и попробуем к нему обратиться
![Alt text](image-8.png)

Получаем ожидаемую ошибку из-за настроенной политики.

Немного погуглив мы находим парочку статьей, рассказывающих нам про интересный байпасс
[click1](https://pulsesecurity.co.nz/advisories/istio-egress-bypass?source=post_page-----c773190e9246--------------------------------)
[click2](https://github.com/DSecurity/istio-security-restrictions-bypass?tab=readme-ov-file#bypass-istio-sidecar)

Это означает, что любой пользователь с uid 1337 может обойти перенаправления траффика на прокси и слать его напрямую.

Мы могли бы сами создать такого пользователя `useradd -u 1337 kacker`, но прочитав /etc/passwd, мы видим, что такой пользователь уже присутствует в системе.
![Alt text](image-9.png)

Логинимся за него `su istio` и пробуем снова курлить наш сервис.
![Alt text](image-10.png)


## Who will guard the guardians?
**Описание:**
*Where pods are being mutated by a foreign regime, one could abuse its bureaucracy and leak sensitive information from the [administrative](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#request) services.*

**Policy:**
```
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: apply-flag-to-env
  namespace: sensitive-ns
spec:
  rules:
    - name: inject-env-vars
      match:
        resources:
          kinds:
            - Pod
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - name: "*"
                env:
                  - name: FLAG
                    value: "{flag}"
```

Заключительный и самый сложный на мой взгляд таск так же знакомит нас с еще одним интересным механизмом - Kyverno.

**Kyverno** - механизм для создания политик, с помощью которых можно проверять, изменять и генерировать ресурсы Kubernetes, обеспечивая точный контроль над их управлением и поведением.

А приведенная выше политика гласит, что каждый pod, созданный в неймспейсе sensitive-ns будет иметь у себя енву с флагом.

Отсюда мы понимаем, что наша цель - создать под в данном namespace и получить заветный флаг.

Попробуем пойти по легкому пути и посмотрим, имеются ли нужные права у нашего сервис аккаунта для создания нового пода.

`kubectl auth can-i create --list`

Конечно же нет
![Alt text](image-11.png)

Как всегда просканим сеть.
![Alt text](image-12.png)
Самым интересным сервисом для нас будет `kyverno-svc.kyverno.svc.cluster.local` 

Для решения нужно разобраться как работают контроллеры допуска.


Когда запрос делается на API сервер Kubernetes, он проходит через несколько этапов, называемых Admission Controllers. После api он переходит к интересующему нас этапу `Mutating admission`, а именно редиректится на веб-хук без аутентификации, в нашем случае это Kyverno.
Если модуль соответствует политике, `apply-flag-to-env`, то он будет "мутировать".

![Alt text](image-13.png)

Как мы выяснили, отправить запрос на создание нужного нам ресурса легитимным способом, не представляется возможным. 

Поэтому наша цель отправить запрос напрямую к Kyverno в обход аутентификации.

Конечно же мы смотрим подсказки, чтобы ненароком не стать психично хворийм и видим, что нам советуют инструмент [kube-review](https://github.com/anderseknert/kube-review)

Данный инструмент поможет нам преобразовать описанный наим куб ресурс в запрос Kubernetes AdmissionReview, который отправится сразу к Kyverno.