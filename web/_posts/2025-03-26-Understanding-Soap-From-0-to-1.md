---
layout: post
author: Alex Abramov
title: Understanding SOAP from 0 to 1
---

# Что такое SOAP?

SOAP это протокол, использующий XML для передачи сообщений, используя HTTP (Также возможно использование других протокол для обмена сообщений, но мы будем говорить конкретно про HTTP/HTTPS).

# Что такое WSDL?

WSDL это схема XML, по которой описывается по каким правилам необходимо создать XML сообщение для взаимодесйтвия с сервисом SOAP. WSDL предоставляет тип протоколов для взаимодействия, структуру сообщения, все возможные операции взаимодействия, различные возможные ответы на сообщения. 

Схема показывает нам как простроить сообщение чтобы впоследствии получить успешный ответ от сервера. Чаще всего wsdl схема будет похожа на болото с бесконечно-непонятным и пугающим описанием. Но если ты сможешь прочитать его однажды - у тебя не будет проблем прочитать любую схему любой сложности.

## Где находится WSDL схема?

Необходимо иметь доступ к API. Например, api калькулятора - [http://www.dneonline.com/calculator.asmx](http://www.dneonline.com/calculator.asmx).

Данное API хорошо побходит для практики и отправки первого запроса через SOAP. У неё есть wsdl схема, для того чтобы найти её нужно добавить `?wsdl/ в конце ссылки ([http://www.dneonline.com/calculator.asmx?wsdl](http://www.dneonline.com/calculator.asmx?wsdl)).

# Защищённость SOAP

## Почему оно безопасно?

SOAP безопасен благодаря строгим стандартам, которые требуют создавать xml документ исключительно по описываемым провилам в схеме. Ты всегда знаешь получаемые данные, различные ответы в успешных и проваленных сценарий. Большинство схем требуют чтобы вы свой запрос подписывали по определённым стандартам безопасности, производили шифрование запроса. Благодаря этому решаются проблемы целостности данных, повторное воспроизведение запросов, подмена сообщений и другие.

## Пример http запроса, использующий SOAP:

```xml
POST /WebService.asmx HTTP/1.1
Host: www.example.com
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://example.com/GetInfo"
Content-Length: length

<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xmlns:xsd="http://www.w3.org/2001/XMLSchema"
               xmlns:soap="http://schemasxmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetInfo xmlns="http://example.com/">
      <param1>value1</param1>
      <param2>value2</param2>
    </GetInfo>
  </soap:Body>
</soap:Envelope>
```

Наше xml-soap-сообщение помещается в тело http-сообщения, и мы также получаем аналогичные http-запросы.

## Из чего состоит SOAP-сообщение

### xml

SOAP сообщение начинается с объявления стандарта. Различные API требует различные версии xml, по этому нам необходимо уточнять серверу по какой версии xml у нас простроено сообщение.  
```xml
<?xml version="1.0" encoding="utf-8"?>
```

### Envelope

Дальше, сообщение должно начинаться с Envelope. Каждое сообщениие должно начинаться именно с этого элемента. Оно представляет структуру и переменные-неймспейсы (namespace).  `xmlns:xsi` `xmlns:xsd` и `xmlns:soap` являются неймспейсами которые стандартизируют наше сообщение. Благодаря этому получатель сообщения сможет понять по каким правилам построено наше сообщение.

```xml
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" # xmls
               xmlns:xsd="http://www.w3.org/2001/XMLSchema"
               xmlns:soap="http://schemasxmlsoap.org/soap/envelope/">
</soap:Envelope>
```

### Header

Header является не обязательным элементов, хранящий в себе метаданные для сообщения, например аутентификацию, информацию об адресе. Часто бывает что Header пустой. Он также может хранить неймспейсы, параметры и другие структуры, также как и на `body`

```xml
# there is empty Header without any data
<soapenv:Header/>

# there is authentication data for accesssing message
<soapenv:Header>
  <ex:Security> # were dont defined "ex" namespace, its just an example.
      <ex:UsernameToken>
        <ex:Username>john_doe</ex:Username>
        <ex:Password>securePassword123</ex:Password>
      </ex:UsernameToken>
  </ex:Security>
</soapenv:Header>
```

### Body

Наше тело SOAP хранит запрос к конкретному ресурсу или методу, а также параметры для ресурсов или методов. Это могут быть различные фильтры, параметры для создание ресурсов и другие вещи для взаимодействия. В случае с калькулятором мы обращаемся к методу (по типу "прибавить", "отнять", "умножить" или "разделить") и оставляем 2 параметра для того, чтобы произвести нам метод на них.  

- `GetInfo` - Это метод к которому мы хотим постучаться.
- `param1` и `param 2` - Это параметры для взаимодействия, хранятся внутри метода.

```xml
<soap:Body>
  <GetInfo xmlns="http://example.com/">
    <param1>value1</param1>
    <param2>value2</param2>
  </GetInfo>
</soap:Body>
```

## Ответ SOAP сообщения

SOAP также возвращает xml документ с похожей структурой, состоящий из `Envelope`, `Body`. Иногда вы можете встретить другой элемент - `Fault`

### Fault

Fault это необязательный элемент, который показывает информацию об ошибках, которые были допущены во время обработки сообщения. Данное сообщение можно встретить когда отправленное сообщение было построено неправильно, имеет ошибки.

```xml
<soapenv:Body>
  <soapenv:Fault>
      <faultcode>soapenv:Client</faultcode>
      <faultstring>Invalid parameters</faultstring>
      <faultactor>http://example.com/service</faultactor>
      <detail>
        <errorCode>1</errorCode>
        <errorMessage>Provided param1 is not valid in the system.</errorMessage>
      </detail>
  </soapenv:Fault>
</soapenv:Body>
```

# Ws-Security

WS-Security - это часть, которая обеспечивает механизмы безопасности для веб-сервисов. Она гарантирует, что сообщение не было изменено во время передачи.

```xml
<soapenv:Header>
  <wsse:Security xmlns:wsse="http://schemas.xmlsoap.org/ws/2002/07/secext">
    <wsse:UsernameToken>
      <wsse:Username>testUser</wsse:Username>
      <wsse:Password>testPassword</wsse:Password>
    </wsse:UsernameToken>
  </wsse:Security>
</soapenv:Header>
```

# Как работать с SOAP и производить его отладку (debug)?

Существует множество инструментов для работы с SOAP, но я рекомендую **SoapUI**. Это ультимативно удобный инструмент, который помогает автоматически генерировать xml-документ и отправить запрос внутри него. Это просто Postman из мира xml. 10/10

Многие конкретно полагаются на SoapUI, компании отправляют свои soap api's с экспортированными схемами soapUI. В один клик вы можете импортировать и получить все методы со всеми параметрами, примеры запросов/ответов.

# SOAP on Ruby

Многие предпочитают строить XML самостоятельно, это один из самых простых способов отправки запросов. возводя огромный монолит ООП, который возвращает строку с XML-документом. Это хороший способ, но вам стоит подумать о поддержке этого огромного куска кода. 

В данном примере нету огромного ООП класса, а наше soap сообщение просто является строкой.

```ruby
require 'net/http'
require 'uri'

soap_message = <<~XML
  <?xml version="1.0" encoding="UTF-8"?>
  <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="http://example.com/">
    <soapenv:Header/>
    <soapenv:Body>
      <web:GetData>
        <web:Parameter>SampleValue</web:Parameter>
      </web:GetData>
    </soapenv:Body>
  </soapenv:Envelope>
XML

url = URI.parse("http://example.com/soap-endpoint")
http = Net::HTTP.new(url.host, url.port)
request = Net::HTTP::Post.new(url.path, { 'Content-Type' => 'text/xml; charset=utf-8' })
request.body = soap_message

response = http.request(request)
puts response.body
```

Но способ выше не всегда подходит, множество SOAP сообщений требуют

# Полезные ruby gems

В Ruby есть несколько гемов, которые пригодятся для решения проблем:

## [Savon](https://github.com/savonrb/savon) - Тяжелый металлический SOAP-клиент, который генерирует xml и отправляет запросы на ruby
У рубина существует множество инструментов для работы с SOAP. Самый популярный из них [savon](https://github.com/savonrb/savon)

```ruby
require 'savon'

# Initialize the client with the WSDL URL of the SOAP service
client = Savon.client(wsdl: 'http://www.dneonline.com/calculator.asmx?WSDL')

# Make a request to the 'Add' operation of the service
response = client.call(:add, message: { 'intA' => 5, 'intB' => 3 })
response.body
```
Гем, используя схему wsdl, создает xml документ по заданным правилам. Фактически сгенерированный xml будет иметь вид:

```xml
<?xml version="1.0" encoding="utf-8"?>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:web="http://www.dneonline.com/webservice/">
   <soapenv:Header/>
   <soapenv:Body>
      <web:Add>
         <web:intA>5</web:intA>
         <web:intB>3</web:intB>
      </web:Add>
   </soapenv:Body>
</soapenv:Envelope>
```

## [XMLDSig](https://github.com/benoist/xmldsig)  

**XMLDSig (XML Digital Signature)** — это стандарт для подписания XML-документов, обеспечивающий их подлинность, целостность и невозможность отказа от подписи. Цель XMLDSig — позволить подписывать XML-документ или его часть, предоставляя доказательство того, что документ не был изменён после подписания.  

Самостоятельное подписание документа — сложный процесс, поэтому существует гем [XMLDSig](https://github.com/benoist/xmldsig), который значительно упрощает задачу.  

Однако есть и плохие новости — гем больше не поддерживается его создателями. Тем не менее, он работает на новых версиях Ruby (в моём проекте успешно используется с Ruby 3.3.0).  

## Конец

**SOAP** — это протокол, который **строго** требует соблюдения определённых правил. Он безопасен и невероятно совместим между разными платформами благодаря стандартам.  

Однако он сложнее, чем другие архитектурные стили (например, REST), поскольку требует тщательного подхода к построению XML-документов, чтобы они соответствовали требованиям валидности.

