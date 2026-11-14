# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Брус Микита Олександрович |
| Група | ІПЗ F2 2.01 |
| Ідентифікатор | s03 |
| Дата виконання |14.11/2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `origin.s03.lab` |
| Імена для сертифіката | `www.s03.lab`, `shop.s03.lab` |
| Ваш сервер (IP) | `10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
server: web
Host: origin.s03.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
05:03:40.405813 IP 10.53.1.1.33620 > 10.53.0.80.80: Flags [P.], seq 1:78, ack 1, win 502, options [nop,nop,TS val 672624616 ecr 322632618], length 77: HTTP: GET / HTTP/1.1
(.o..:..GET / HTTP/1.1
Host: origin.s03.lab
server: web
Host: origin.s03.lab
```

Вивід кроку 1.2:

```text
server: web
Host: origin.s03.lab
SNI: origin.s03.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
0
1
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | GET/HTTP/1.1 | 0 сходженнь, не видно, тому що, воно зашифровано |
| заголовок `Host` | Host: origin.s03.lab | Також зашифровано |
| тіло відповіді (`server: web`) | server: web | Також зашифровано |
| ім'я віддаленого сервера (кількість входжень) | 2, а саме в хост (2 строка) ну і в тілі відповіді | 1, чому? Тому що, щоб сервер зрозумів який сертифікат йому дістати, браузер за допомогою Client hello(першому пакеті), передає ім'я сайта в поле SNI. Це все відбувається до початку шифрування |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?
Як я і писав вище, воно передається в першому пакеті ClientHello до поля SNI(Server Name Indication)
2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?
HTTP не має рукостискання TLS, там не існує SNI. А у HTTPS, у нього як раз є Рукостискання TLS, де клієнт вже і передає SNI, щоб сервер зрозумів, який треба сертифікат
3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?
Під час TLS Рукостискання, браузер і сервер череез ALPN(Application-Layer-Protocol-Negotiation, це розширення для TLS) сразу домовились, що раз вони оба вміють через http2, то вони будуть цим користуватися. А в обичному http взагалі нема рукостискання, тому там по замовчуванням 1.1 http
## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=325
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=979
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
```

Вивід кроку 2.2 (TLS 1.2):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=220
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=974
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 | К до С | Handshake (22) | - | ClientHello | 325 |
| 2 | С до К | Handshake (22) | - | ServerHello | 118 |
| 3 | С до К | ChangeCipherSpec (20) | - | ChangeCipherSpec | - |
| 4 | С до К | ApplicationData (23) | Handshake (22) | EncryptedExtensions | 15 |
| 5 | С до К | ApplicationData (23) | Handshake (22) | Certificate | 979 |
| 6 | С до К | ApplicationData (23) | Handshake (22) | CertificateVerify | 75 |
| 7 | С до К | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 8 | К до С | ChangeCipherSpec (20) | - | ChangeCipherSpec | - |
| 9 | К до С | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 10 | К до С | ApplicationData (23) | Alert (21) | Alert | - |

Вивід кроків 2.3 і 2.4:

```text
extension_type=server_name(0), length=19
          0000 - 00 11 00 00 0e 6f 72 69-67 69 6e 2e 73 30 33   .....origin.s03
          000f - 2e 6c 61 62                                     .lab
        extension_type=ec_point_formats(11), length=4
TLS 1.3: 0
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | 325 | 220 |
| Номер першого запису, тип якого спостерігач не бачить |4, зовнішній тип під ApplicationData (23), справжній тип всередині Inner Content Type = Handshake (22) | 8, запис Finished сразу після ChangeCipherSpec клієнта |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 | 2 |
| Чи видно сертифікат у дампі (результат 2.4) | Ні | Так, Ім'я, яке передається відкрито |
| Обраний протокол ALPN | h2 | h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
Шифрування починається саме с 10 рядочку та 11, Content Type = ApplicationData (23) та Inner Content Type = Handshake (22), це в нас четвертий запис. Для людини, яка захоче подивитися записи, все виглядає однаково, для нього це чи рукостискання, чи графік http, він його не бачить благодоря шифруванню 
2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
В версії 1.3, клієнт сразу відсилає параметри для обміна ключами в першому пакеті ClientHello за допомогою key_share( це розширення для 1.3 TLS). 
3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
5 службових байт + довжина доменного імені, 2 байти ена довжину списку імен, 1 байт на тип імені, 2 байти на довжину самого імені. Для мого сервера origin.s03.lab, (14 символів = 14 байт) це 14+5=19 Байт
4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
Це для старого обладнання, вони якщо не зустрінуть ChangeCipherSpec, вони можуть розірвати з'єднання, тому що, звикли працювати з 1.2 версією
## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 549991
notBefore=Nov 14 23:19:22 2026 GMT
notAfter=Nov 14 23:19:22 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
More details here:https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
2026/11/14 23:24:27 [INFO] [www.s03.lab] acme: Trying to solve HTTP-01
2026/11/14 23:24:29 [INFO] [www.s03.lab] The server validated our request
2026/11/14 23:24:29 [INFO] [shop.s03.lab] acme: Trying to solve HTTP-01
2026/11/14 23:24:34 [INFO] [shop.s03.lab] The server validated our request
2026/11/14 23:24:34 [INFO] [www.s03.lab, shop.s03.lab] acme: Validations succeeded; requesting certificates
2026/11/14 23:24:34 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/14 23:24:34 [INFO] [www.s03.lab] Server responded with a certificate.
10.53.0.10 - - [14/Nov/2026:23:24:27 +0000] "GET /.well-known/acme-challenge/t31zVkZaRjylukvcuWO0LfJAugV_Isc3fppL-TlKAII HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:23:24:27 +0000] "GET /.well-known/acme-challenge/t31zVkZaRjylukvcuWO0LfJAugV_Isc3fppL-TlKAII HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:23:24:27 +0000] "GET /.well-known/acme-challenge/t31zVkZaRjylukvcuWO0LfJAugV_Isc3fppL-TlKAII HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:23:24:30 +0000] "GET /.well-known/acme-challenge/ltxew5UU-XDkNHLECsT-EdJYhrfOsvmr0MYcXSYpQLg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:23:24:30 +0000] "GET /.well-known/acme-challenge/ltxew5UU-XDkNHLECsT-EdJYhrfOsvmr0MYcXSYpQLg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:23:24:30 +0000] "GET /.well-known/acme-challenge/ltxew5UU-XDkNHLECsT-EdJYhrfOsvmr0MYcXSYpQLg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
subject=CN = www.s03.lab
issuer=CN = Pebble Intermediate CA 296c35
notBefore=Nov 14 23:24:34 2026 GMT
notAfter=Nov 14 23:24:33 2031 GMT
serial=0812DFBACED0A4A3
X509v3 Subject Alternative Name: 
    DNS:www.s03.lab, DNS:shop.s03.lab
subject=CN = Pebble Intermediate CA 296c35
issuer=CN = Pebble Root CA 549991
```

Вивід кроку 3.5:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/14 23:25:35 [notice] 47#47: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here:https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.s03.lab
   i:CN = Pebble Intermediate CA 296c35
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 14 23:24:34 2026 GMT; NotAfter: Nov 14 23:24:33 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
server: s03 (https)
Host: shop.s03.lab
SNI: shop.s03.lab
protocol: TLSv1.3
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | CN = Pebble Intermediate CA 296c35 |
| Видавець сертифіката видавця | CN = Pebble Root CA 549991 |
| Імена в SAN | DNS:www.s03.lab, DNS:shop.s03.lab |
| Строк дії (від – до) | Nov 14 23:24:34 2026 GMT - Nov 14 23:24:33 2031 GMT |
| Тип ключа сервера і алгоритм підпису | Ключ: id-ecPublicKey (256 біт), підпис видавця: RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | У 3.5: 1 сертифікат; у 3.6: 2 сертифіката |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
Це в нас перевірив центр сертифікації ACME. Як саме? Він застосував метод HTTP-01, Клієнт LEGO розмістив скриті токени(файл) по путі /var/www/acme/.well-known/acme-challenge, а після цього, центр сертифікації зробив запити HTTP к нашому серверу nginx для двоїх імен, і получив код 200(ОК). Ось рядки, які це підтверджують 10.53.0.10 - - [14/Nov/2026:23:24:27 +0000] "GET /.well-known/acme-challenge/t31zVkZaRjylukvcuWO0LfJAugV_Isc3fppL-TlKAII HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"10.53.0.10 - - [14/Nov/2026:23:24:30 +0000] "GET /.well-known/acme-challenge/ltxew5UU-XDkNHLECsT-EdJYhrfOsvmr0MYcXSYpQLg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"[www.s03.lab] The server validated our request і ще [shop.s03.lab] The server validated our request 
2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?
`--cacert` в нього є тільки корневий сертифікат, (Pebble Root CA 549991), а сертифікат нашого сайту має підпис тільки промижного центра (Pebble Intermediate CA 296c35). Сам сервер надіслав тільки свій сертифікат, то есть, без проміжного, із-за цього, клієнт не зміг зробити повний ланцюжок. Тут видно (Verify return code: 21 (unable to verify the first certificate)) 
3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?
id-ecPublicKey, це наш ключ сервера, а RSA-SHA256, це яким способом, центр сертифікації підписав сертифікат. Чому не суперечать? Тому що, це різні речі. перший відповідае за ключ нашого сервера, а другий вже за підпис центру сертифікації 
4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?
Pebble просто створить нові корневі та промежні ключі, із-за цього, старий сертифікат перестане проходити перевірку. А якщо в житті таке буде, то тоді старі сертифікати можуть перестати проходити перевірки(Але не завжди, якщо змінити там корневий ключ, то старі сертифікати не завжди можуть перестати проходити перевірку) 
## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
=== f1.s03.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f1.s03.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Apr 28 04:55:55 2026 GMT; NotAfter: Oct 25 04:55:55 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Nov 13 04:55:55 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f2.s03.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.s03.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Feb 12 04:55:55 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Nov 13 04:55:55 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f3.s03.lab
server: web, f3
Certificate chain
 0 s:CN = f3.s03.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Feb 12 04:55:55 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Nov 13 04:55:55 2031 GMT
---
Verify return code: 0 (ok)
=== f4.s03.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f4.s03.lab
   i:CN = f4.s03.lab
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Feb 12 04:55:55 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f5.s03.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f5.s03.lab'
Certificate chain
 0 s:CN = legacy.s03.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Feb 12 04:55:55 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 04:55:55 2026 GMT; NotAfter: Nov 13 04:55:55 2031 GMT
---
Verify return code: 0 (ok)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | certificate has expired | 10 (certificate has expired) | Срок дії сертифіката пройшов | v:NotBefore: Apr 28 04:55:55 2026 GMT; NotAfter: Oct 25 04:55:55 2026 GMT | обновити прострочений сертифікат у центрі сертифікації |
| f2 | unable to get local issuer certificate | 20 (unable to get local issuer certificate) | Перевірка довіри до кореневого центру сертифікації | 1 s:O = Foreign Org, CN = Foreign Intermediate CA та i:O = Foreign Org, CN = Foreign Root CA | Отримати сертифікат від довіреного центру або добавити Foreign Root CA у довірені корені клієнта |
| f3 | Немає (справний) | 0 (ok) | Усе пройдено успішно | server: web, f3 та Verify return code: 0 (ok) | Нічого міняти не треба |
| f4 | self-signed certificate | 18 (self-signed certificate) | сертифікат самопідписаний | 0 s:CN = f4.s03.lab та i:CN = f4.s03.lab | Замінити самопідписаний сертифікат на підписаний довіреним ЦС обов язково |
| f5 | no alternative certificate subject name matches target host name 'f5.s03.lab' | 0 (ok) | Не правильне доменне ім я (його нема в SAN/CN) | 0 s:CN = legacy.s03.lab | Перевипустити сертифікат, добавивши f5.s03.lab у поле SAN (Subject Alternative Name) |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?
На сервері f2.s03.lab сертифікат видан центром, якому клієнт не довіряє (Foreign Root CA). Це в нас видно по виводу openssl s_client: 1. сервер надсилає два сертифікати у ланцюжку (0 і 1), проміжний Foreign Intermediate CA є, но он нас веде до невідомого корня Foreign Root CA, і при цьому вибиває код помилки 20 (unable to get local issuer certificate). 2. А якщо дивитися на наш крок 3.5, де сервер не надіслав проміжний сертифікат, то там у виводі є тільки один сертифікат (0), а код помилки вже 21 (unable to verify the first certificate).
2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.
Є в мене такий сервер, f5.s03.lab(сертифікат legacy.s03.lab), Сама команда openssl s_client, вона перевіряє тільки правильність крипто-подпису, строк дії ті ланцюжок довіро до кор.центру. А з цим, тут все добре, повертають 0(ок). А вот curl після провірки ланцюжка звіряє ім я домену з запиту URL з полями SAN/CN сертифіката и блокує з єднання через розбіжність
3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?
Всі сервера відповіли б. Сам ключ -k вбирає у нас саму перевірку справжності сертифіката, і якщо порівняти його з --cacert, то тут вже пропадає автентифікація сервера, з єднання ще зашифроване, но у клієнта нема гаранті, що він говоре з справжнім сервером.
## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 1 RTT | 1 RTT | 0 RTT |
| `time_appconnect` | 2 RTT | 3 RTT | 0 RTT |
| `time_starttransfer` | 3 RTT | 4 RTT | 1 RTT |

Вивід кроків 5.2 і 5.3:

```text
0.110358 0.180621 0.253589 0.289117 0.289176 2 1
0.035476 0.070623 0.109212 0.144542 0.144594 2 1
0.035441 0.070559 0.109436 0.144768 0.144834 2 1
0.035421 0.070532 0.144777 0.180096 0.180158 2 1
0.035484 0.070671 0.144394 0.179751 0.179822 2 1
0.035477 0.070639 0.144081 0.179458 0.179517 2 1
0.035576 0.070818 0.110210 0.145548 0.145614 2 1
0.000018 0.000000 0.000000 0.035537 0.035595 2 0
0.000018 0.000000 0.000000 0.035337 0.035397 2 0
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 35.5 | 70.6 | 109.4 | 144.8 | 144.8 | 1 |
| TLS 1.2, нове | 35.5 | 70.6 | 144.4 | 179.8 | 179.8 | 1 |
| 5.3, перший запит | 35.6 | 70.8 | 110.2 | 145.5 | 145.6 | 1 |
| 5.3, другий запит | 0.02 | 0 | 0 | 35.5 | 35.6 | 0 |
| 5.3, третій запит | 0.02 | 0 | 0 | 35.3 | 35.4 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено: 35.4 мс (це в нас за `starttransfer` повторних запитів у 5.3: 35.54 мс і 35.34 мс, ну і также за `connect` - `namelookup` = 70.62 - 35.48 = 35.14 мс, де затримка netem для мого варіанта s03 = 35 мс)

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?
Якщо DNS є в кеші (namelookup = 1 RTT, десь 35.5 мс):
Для TLS 1.3 (нове): connect=2 RTT (70.6 мс), appconnect=3 RTT + 3.5 мс (109.4 мс), starttransfer = 4 RTT + 3.5 мс (144.8 мс). У першому холодному замірі, кеш ще був пустий, тому namelookup взяв собі 3 RTT (110.4 мс) і по ітогу вийшло 6 RTT.
Для TLS 1.2 (нове): connect=2 RTT (70.6 мс), appconnect=4 RTT + 3.0 мс (144.4 мс), starttransfer = 5 RTT + 3.0 мс (179.8 мс).
Для повторного з єднання (крок 5.3): namelookup, connect і appconnect по 0 RTT (0-0.02 мс), а starttransfer = 1 RTT (35.3-35.5 мс).
Різниця від цілого числа RTT на етапі appconnect (набігає лишніх 3-3.5 мс). Чому, тому що, ну RTT в нас, це час, поки пакети летят по мережі, а під час TLS рукостискання, проц тратить час на генерацію ключей і перевірка сертифіката
2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?
Тому що резолвер (10.53.0.53) знаходиться в другому контейнері, запит до нього все одно проходить через мережу, де є затримка netem(35.5мс). Час був би десь в два рази більше (71 мс), якщо резолверу би не вистачало запису в кеші й йому довелося зробити додатковий запрос до другого DNS сервера
3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.
Мій середній RTT=35.4 мс
Якщо кожен запит використовує нове з єднання TLS 1.3: виходить 4 RTT на запит (DNS+TCP+TLS+HTTP). 200 *4*35.4 = 28 320 мс = 28.3 секунди
Для кожного нового з єднання TLS 1.2: виходить 5 RTT на запит (DNS + TCP + 2 TLS + HTTP). Рахуємо: 200*5*35.4 = 35 400 мс = 35.4 секунди
Якщо всі 200 запитів йдуть через одне з єднання: перший запит відбирає 4 RTT, а останні 199, по 1 RTT. Разом уе в нас 203 RTT: 203*35.4 = 7 186.2 мс = 7.2 секунди
## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно: Несподіваним було внезапне попередження у 3.3 (під час запуску команди lego для випуску сертифіката), де вилізла рамка з написом !!!! HEADS UP !!!! та Your account credentials have been saved. Я думав побачити просто звичайні рядки про отримання сертифіката, забавно

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: Так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Gemini Hydrogen | Строка про RTT, ответы на три вопроса, задание пять. Скажи, как он, соответствует, не соответствует правде | Перевірка правильності власних розрахунків RTT та часу очікування мережі для 200 запитів у завданні 5 |
| 2 | Gemini Hydrogen | Почему при одинаковом запросе curl в задании 1.2 сервер показал protocol: HTTP/2.0 TLSv1.3, а в задании 3.6 мой сервер для shop.s03.lab показал просто protocol: TLSv1.3? | Пояснення роботи розширення ALPN та налаштувань HTTP/2 у веб-сервері nginx після завершення роботи |

