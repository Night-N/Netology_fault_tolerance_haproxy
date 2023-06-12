# Кластеризация и балансировка нагрузки
## Домашнее задание. Горбунов Владимир

## Цель задания
1. Настраивать балансировку с помощью HAProxy
2. Настраивать связку HAProxy + Nginx

## Содержание

- [Задание 1. HAPROXY level 4 + simple python server](#Задание-1)
- [Задание 2. HAPROXY level 7 + simple python server](#Задание-2)  
- [Задание 3. HAPROXY + nginx + simple python server](#Задание-3) 
- [Задание 4. HAPROXY. Два бекэнда, simple python server](#Задание-4)  


## Задание 1
> Запустите два simple python сервера на своей виртуальной машине на разных портах

> Установите и настройте HAProxy, воспользуйтесь материалами к лекции по ссылке

> Настройте балансировку Round-robin на 4 уровне.

> На проверку направьте конфигурационный файл haproxy, скриншоты, где видно перенаправление запросов на разные серверы при обращении к HAProxy.

- Решил настроить локальный simple python server с https.
1. Необходимо было создать локальный сертификат для доменов, на которых будет доступен сервер \(example.local, localhost, 127.0.0.1\) </br>
Для этого воспользовался утилитой mkcert (https://github.com/FiloSottile/mkcert), которую удобно применять для  создания самоподписных сертификатов, добавления собственного корневого сертификата в системное хранилище и в хранилище сертификатов браузеров Firefox и Chrome на время разработки.
> git clone https://github.com/FiloSottile/mkcert && cd mkcert

> go build -ldflags "-X main.Version=$(git describe --tags)"

> ./mkcert  example.local localhost 127.0.0.1 ::1

> ./mkcert -install

2. Запущены два Python сервера, которые осуществляют финализирование SSL трафика. Дополнительно отключено кэширование, чтобы при обращении к localhost:443 было наглядно видно переключение между серверами. Сервера подняты на портах 8001 и 8002.
```python
import http.server
import socketserver
import ssl

class MyHTTPRequestHandler(http.server.SimpleHTTPRequestHandler):

    def end_headers(self):
        self.send_my_headers()
        http.server.SimpleHTTPRequestHandler.end_headers(self)

    def send_my_headers(self):
        self.send_header("Cache-Control", "no-cache, no-store, must-revalidate")
        self.send_header("Pragma", "no-cache")
        self.send_header("Expires", "0")


if __name__ == '__main__':
    address = ('', 8001)
    httpd = socketserver.TCPServer(address, MyHTTPRequestHandler)
    httpd.socket = ssl.wrap_socket(httpd.socket,
                               server_side=True,
                               certfile="server.pem",
                               keyfile="key.pem",
                               ssl_version=ssl.PROTOCOL_TLS)
    httpd.serve_forever()

```
3. Haproxy 
   - Настроен фронтэнд на 443 порту, в режиме tcp
   - Для корректного перенаправления https трафика проверяется наличие заголовка req_ssl_hello_type 1 со стороны клиента и трафик предназначенный для localhost перенаправляется на бекэнд сервер.
   - Бекэнд работает в режиме tcp, roundrobin
   - Haproxy в данном примере не финализирует SSL, при этом должен корректно обрабатывать его 
   - Включена sticky table, в которой haproxy хранит и отслеживает сессии
   - Добавлен acl для отслеживания SSL трафика от клиента к серверу req_ssl_hello_type 1 и ответы сервера клиенту rep_ssl_hello_type 2
   - Входящие tcp запросы принимаются от acl clienthello, a ответы от acl serverhello
   - В процессе установления SSL соединения Haproxy анализирует трафик по полю SSL Client Hello message. Запись и чтение состояния сессии осуществляется по 43му байту http заголовка. Запись состояния сессии осуществляется с помощью команды stick on payload_lv(43,1) при наличии clienthello, а запись ответа сервера сохраняется с помощью команды stick store-response payload_lv(43,1) при наличии serverhello.
   - Наконец, опция ssl-hello-chk указывает Haproxy проверять целостность SSL соединения.

```haproxy
frontend py_1
    bind *:443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    use_backend py_servers if { req_ssl_sni -i localhost }
backend py_servers
    mode tcp
    balance roundrobin
    stick-table type binary len 32 size 30k expire 30s
    acl clienthello req_ssl_hello_type 1
    acl serverhello rep_ssl_hello_type 2
    tcp-request inspect-delay 5s
    tcp-request content accept if clienthello
    tcp-response content accept if serverhello
    stick on payload_lv(43,1) if clienthello
    stick store-response payload_lv(43,1) if serverhello
    server py1 127.0.0.1:8001
    server py2 127.0.0.1:8002
    option ssl-hello-chk

```




## Задание 2
> Запустите три simple python сервера на своей виртуальной машине на разных портах

> Настройте балансировку Weighted Round Robin на 7 уровне, чтобы первый сервер имел вес 2, второй - 3, а третий - 4

> HAproxy должен балансировать только тот http-трафик, который адресован домену example.local

> На проверку направьте конфигурационный файл haproxy, скриншоты, где видно перенаправление запросов на разные серверы при обращении к HAProxy c использованием домена example.local и без него.

- Текст:
Текст

- Текст:
Текст

![](img/1.jpg)

- Текст:

[keepalived.conf](./keepalived.conf)



## Задание 3

> Настройте связку HAProxy + Nginx как было показано на лекции.

> Настройте Nginx так, чтобы файлы .jpg выдавались самим Nginx (предварительно разместите несколько тестовых картинок в директории /var/www/), а остальные запросы переадресовывались на HAProxy, который в свою очередь переадресовывал их на два Simple Python server.

> На проверку направьте конфигурационные файлы nginx, HAProxy, скриншоты с запросами jpg картинок и других файлов на Simple Python Server, демонстрирующие корректную настройку.

- Текст:
Текст

- Текст:
Текст

![](img/1.jpg)

- Текст:

[keepalived.conf](./keepalived.conf)


## Задание 4
> Запустите 4 simple python сервера на разных портах.

> Первые два сервера будут выдавать страницу index.html вашего сайта example1.local (в файле index.html напишите example1.local)

> Вторые два сервера будут выдавать страницу index.html вашего сайта example2.local (в файле index.html напишите example2.local)

> Настройте два бэкенда HAProxy

> Настройте фронтенд HAProxy так, чтобы в зависимости от запрашиваемого сайта example1.local или example2.local запросы перенаправлялись на разные бэкенды HAProxy

> На проверку направьте конфигурационный файл HAProxy, скриншоты, демонстрирующие запросы к разным фронтендам и ответам от разных бэкендов.


