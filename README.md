
## Запуск и настройка Nexus как Docker репозитория
### Установим Docker
Официальная инструкция по установке Docker (*в данном контексте для Debian*) https://docs.docker.com/engine/install/debian/

### Запустим Nexus в docker контейнере
```
docker run -d -p 8081:8081 -p 8082:8082 --name nexus -v nexus-data:/nexus-data --restart unless-stopped sonatype/nexus3:3.75.1
```
> **Порты**
>
> 8081 - Основной порт для **UI Nexus**
>
> 8082 - Порт который будет слушать **Docker** репозиторий
>
> **Параметры**
>
> **--restart unless-stopped** - контейнер при случайной остановке будет всегда перезапущен за исключением ручной остановки **docker stop**
>
> **-v nexus-data:/nexus-data** - создаёт и монтирует volume в docker контейнер для сохранения состояние **Nexus**

### Смотрим log запуска docker контейнера Nexus
Дожидаемся запуска и надписи **Started Sonatype Nexus**
```
docker logs -f nexus
```
### Получаем пароль от админской учетной записи
Подключимся к контейнеру с **Nexus**
```
docker exec -it nexus /bin/bash
```
Заглянем в файл **admin.password**
```
cat /nexus-data/admin.password
```
### Настроим Docker репозиторий
После первого входа под пользователем **admin** и полученным ранее сгенерированным паролем система запросит смену пароля

#### Создадим репозиторий docker
1. Нажимаем шестерёнку
2. Create repository
3. docker (hosted)
4. В поле **Name** указваем имя **docker**
5. Проверяем что отмечен чекбкс **If checked, the repository accepts incoming requests**
6. Ставим чекбокс **HTTP** и указываем ранее проброшенный порт **8082** что будем использовать для docker запросов
7. Жмём кнопку в низу **Create repository**

#### Включим docker авторизацию
1. Слева жмём **Security > Realms**
2. Перетаскиваем из колонки **Available** в колонку **Active** компонент **Docker Bearer Token Realm**
3. Внизу жмём **Save**

## Настройки для использования Nginx Reverse Proxy
### Настройки location для Nginx
В моём случае **Nginx Reverse Proxy** реализован в **kubernetes** кластере, поэтому **SSL** терминируется на **Ingress-контрллере**, соответственно в конфигурации с location нет ничего про **SSL** так как он указан у меня в **Ingress Nginx**

**location /v2** предназначен для запросов от docker клиента которые будут заворачиваться на порт **8082** в свою очередь прослушиваемые репозиторием docker созданным ранее в **Nexus**

```
server {
    listen 80;

    server_name nexus.poletaevlev.ru;
    
    location / {
        proxy_pass http://192.168.88.32:8081;
        proxy_http_version  1.1;
        proxy_cache_bypass  $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port  $server_port;
    }
    location /v2/ {
        proxy_pass http://192.168.88.32:8082;
        proxy_http_version  1.1;
        proxy_cache_bypass  $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port  $server_port;
        client_max_body_size 5000M;
    }
}
```
### Настройки Ingress Nginx
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  namespace: {{ .Values.namespace }}
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "5000m"
    nginx.ingress.kubernetes.io/proxy-set-headers: |
      Authorization: $http_authorization
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - nexus.poletaevlev.ru
    secretName: lewc-poletaevlev-ru
  rules:
  - host: nexus.poletaevlev.ru
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .Release.Name }}-nginx
            port:
              number: 80
```

## Проверим работу **Docker репозитория**
##### Авторизуемся в Docker репозитории
```
docker login nexus.poletaevlev.ru
```
##### Перетэгируем образ
```
sudo docker tag image:tag nexus.poletaevlev.ru/image:tag
```
##### Запушим образ в docker репозиторий
```
docker push nexus.poletaevlev.ru/image:tag
```