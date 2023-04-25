# Answers

DevOps

### Задача 1

**Вопрос:** Как проверить, открыт или закрыт порт на удаленном хосте, локальном хосте?

**Ответ:** - 
На удаленном telnet [HostName or IP] [PortNumber] или nmap [-options] [HostName or IP] [-p] [PortNumber] \
telnet точечно, мне удобнее nmap т.к. можно сканировать диапозоны IP/портов \
на локальном - netstat -plunt, в удобной форме

### Задача 2

В Ubuntu есть файл с содержимым:

\# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)

HISTSIZE=1000
HISTFILESIZE=2000

**Вопрос:** С помощью какой команды можно найти данный файл в системе?

**Ответ:** - 
```
grep -lir 'for setting history length see HISTSIZE and HISTFILESIZE in bash(1)' --exclude-dir=proc / 
```
Возьмём самый жирный кусок из содержимого, исключим proc, как виртуальную файловую систему, иначе grep никогда не закончит, находим .bashrc

### Задача 3

**Вопрос:** Объясните разницу между script1 & script2 и script1 && script2, а также script1 && script2 || script3?

**Ответ:** - 
- script1 & script2- амперсанд запускает процесс в бэкграунде, таким образом мы запускаем script2 не ожидаясь окончания выполнения script1 \
- script1 && script2 - AND - последовательное выполнение script1, затем script2. если script1 завершается с ошибкой, script2 не запускается \
- script1 && script2 || script3 - OR - выполняем последовательно script1 и script2, в случае неуспеха запускается script3

### Задача 4

В папке большое количество файлов. 

**Вопрос:** Какой командой можно удалить все файлы в папке, если rm -f ./* завершается с ошибкой Argument list too long?

**Ответ:** - 
тут упираемся в ARG_MAX \
никогда не сталкивался, но задача популярная \
самое простое удалить директорию и воссоздать потом, если надо: rm -r /path/to/directory/ \
через find: find . -type f -delete \
или передав аргумент rm: ls | xargs rm 

### Задача 5

Host1 находится в подсети 172.100.0.0/24 с Host2. Host2 также имеет доступ через VPN к подсети 10.0.0.0/8, в которой находится Host3. 

В подсети 172.100.0.0/24 добавлен маршрут к 10.0.0.0/8 через gateway Host2.

Сетевые интерфейсы на хостах:

- Host1: 172.100.0.5
- Host2: 172.100.0.30, 10.0.0.30
- Host3: 10.0.0.145

При проверке доступа выяснилось, что с Host1 доступен 10.0.0.30, но любые обращения к 10.0.0.145 завершаются с ошибкой по тайм-ауту. 

**Вопрос:** В чем проблема и как решить?

**Ответ:**
Host3 -- обычный хост, который видит только в пределах своей 10.0.0.0 подсети, а значит он видит себя и Host2 (интерфейс 10.0.0.30), имеет default gateway 10.0.0.1 например \
Host1 посылает пакеты к 10.0.0.145 через шлюз 10.0.0.30, пакеты уходят, но ответа нет, т.к. нет обратного маршрута \
Host3 ничего не знает про сеть 172.100.0.0/24, используя свой default gateway посылает вникуда. Надо добавить маршрут.

### Задача 6

Необходимо оптимизировать Dockerfile. **Объяснить**, что и почему так:
```
FROM slim:7.3 
COPY . /app 
RUN curl -sS [https://getcomposer.org/installer](https://getcomposer.org/installer) | php -- --install-dir=/usr/local/bin --filename=composer 
RUN apk add git 
RUN composer install
RUN chown -R www-data:www-data /app
```

**Ответ:**
Если ничего не добавлять, а работать с тем что есть, то так:

FROM slim:7.3 \# тут всё ок, начало докерфайла, слим или алпайн образы рекомендуемы как минималистчиные \
RUN apk add git && curl -sS [https://getcomposer.org/installer](https://getcomposer.org/installer) | php -- --install-dir=/usr/local/bin --filename=composer \
\# тут во-первых уменьшаем колличество слоев, получается один слой установки нужных утилит, и так-же наименее изменяемые слои ставим повыше, чтоб они часто были в кеше \
COPY . /app \# из 3х оставшихся слоёв этот очевидно первый, копируем содержимое текущей папки в app \
RUN composer install \# немного странно учитывая что composer install запустится в текущей директории, а WORKDIR не менялся, предположу что composer install -d /app - устанавливает необходимые пакеты для проекта, складывает в папку vendor \
RUN chown -R www-data:www-data /app \# приложение ориентировано на работу с веб-сервером, поэтому рекурсивно меняем юзера и группу на www-data \

Добавил бы еще apk update \
composer можно копировать: COPY --from=composer /usr/bin/composer /usr/local/bin/composer \
WORKDIR /app

### Задача 7

Deployment strategies. Какие существуют и чем отличаются (recreate, blue-green, canary etc.)?

**Ответ:**
- Recreate - Стратегия при которой старая версия ПО выключается, затем включается новая. 
	- \+ дешево,т.к. всегда предполагает одну работающую версию ПО. приложение обновляется полностью 
	- \- главный минус это даунтайм, пока выключается старая версия, включается новая. поэтому подходит только для тестовых сред 
- Ramped - он же rolling upgrade. Постепенное,поэтапное обновление версии путем добавления новых экземпляров ПО, удаления устарых, до тех пор пока не останутся только новые. 
	- \+ прозрачно для пользователя. без даунтайма 
	- \- может быть затратно по времени. нет контроля над трафиком 
- Blue/Green - новая версия ПО(Green) разворачивается параллельно со старой, стабильной(Blue). Тестируется, Если всё ок ЛБ переключаем всё на Green 
	- \+ Мгновенное переключение между версиями за счет ЛБ, откат в случае проблем. Нет проблем с пересечением версий. 
	- \- удваивает ресурсы, т.к. надо держать и старую версию и новую одновременно. 
- Canary - 	Стратегия заключается в постепенном перемещении траффика со старой версию на новую за счет ЛБ, процентного соотношения базирующегося на весе. 
    - \+ Удобен для контроля над ошибками новой версии. быстрый откат за счет ЛБ 
	- \- За счет постепенного смещения траффика медленное развертывание 
- A/B testing - предпологает развернутую новую версию(или несколько), доступ к которой осуществляется за счет ЛБ при определенных условиях (языка, геопозиции, ОС, параметрам запроса и т.д.) .В основном для сбора статистики, но основе которой делаются бизнес-решения. Либо выбор версии на основе конверсии 
	- \+ Несколько версий в параллели, контроль над трафиком 
	- \- ЛБ требует серьёзной настройки 
- Shadow - предпологает развернутую и старую и новыю версии	(Shadow), при это какая то часть запросов (копия) отправляется на Shadow для проверки 
	- \+ Тестирование на основе реально продакшн траффика. Остается в тени пока стабильность и производительность не будут соответствовать требованиям 
	- \- Опять же дорого, двойные ресурсы. Сложность в настройке, как следствие вероятность ошибки и ввод реальных пользователей в заблуждение. 

### Задача 8

Подготовить helm chart для приложения: 
[https://gitlab.com/zedtux/docker-coming-soon](https://gitlab.com/zedtux/docker-coming-soon). Можно переслать архивом или ссылкой на публичный git-репозиторий.

**Ответ:**
https://github.com/MrBochanov/helm-coming-soon  

### Задача 9

Запустить локально minikube и настроить Ingress DNS ([https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/](https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/)). Затем выполнить тестовые задания:

Запустил, настроил ingress-dns c dnsmasq

**Задание 1:** 

Развернуть в локальном minikube (с рабочим Ingress DNS) проект [https://gitlab.com/hdrus3/check1](https://gitlab.com/hdrus3/check1). Проект доступен локально по URL: [http://sim2.test/](http://sim2.test/).
Клиент пожаловался на ошибочное поведение страницы [http://sim2.test/ping](http://sim2.test/ping). Необходимо найти и исправить ошибку, объяснить причину.

**Ответ:**
Deployment helloworld создаёт ReplicaSet который создаёт 2 пода совпадающих с лейблом  project: helloworld \
Темплейт project: helloworld соответствует одному контейнеру, базирующемуся на testcontainers/helloworld с портом 8080 \

Deployment (name: helloworld-on-port-80) создаёт ReplicaSet, который создёт 1 под совпадающий с лейблом  project: helloworld и test: helloworld-on-port-80 \
Темплейт project: helloworld и test: helloworld-on-port-80 соответствует одному контейнеру, базирующемуся на olliefr/docker-gs-ping с портом 8080 \

Service (name: helloworld) таргетит 8080 порт к любому поду project: helloworld 8080 порту \
Service (name: helloworld-on-port-80) таргетит 8080 порт к любому поду test: helloworld-on-port-80 8080 порту \

Таким образом у нас все три пода имеют лейбл project=helloworld, являясь ендпоинтами сервиса helloworld: 
```
bocha@Legion:~/Projects/check/check1$ kubectl describe svc helloworld
Name:              helloworld
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          project=helloworld
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.107.83.176
IPs:               10.107.83.176
Port:              default  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.244.0.24:8080,10.244.0.25:8080,10.244.0.26:8080
Session Affinity:  None
Events:            <none>
```

Но только 2 из них могут ответить 200 на /ping \

Решить можно по разному, самое наверное очевидное - переделать application-2.yaml \
либо убрать лейбл project: helloworld из Deployment, либо задать project: helloworld-on-port-80, и в сервисе тоже \
https://github.com/MrBochanov/check/tree/main/check1

**Задание 2:**

Развернуть в локальном minikube (с рабочим Ingress DNS) проект [https://gitlab.com/hdrus3/check2](https://gitlab.com/hdrus3/check2), состоящий из двух web-приложений.

Необходимо обеспечить доступность этих приложений по URL [http://simulation3.test/](http://simulation3.test/)

При этом:

- [http://simulation3.test/ping](http://simulation3.test/ping) должно обрабатывать приложение helloworld;
- [http://simulation3.test/uuid](http://simulation3.test/uuid) должно обрабатывать приложение helloworld;
- весь остальной трафик должно обрабатывать приложение docker-gs-ping.

**Ответ:**
Не понял подвоха, просто написал ingress, работает
https://github.com/MrBochanov/check/tree/main/check2
