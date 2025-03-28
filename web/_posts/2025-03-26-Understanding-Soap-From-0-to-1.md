---
layout: post
author: Alex Abramov
title: Understanding SOAP from 0 to 1 (using ruby)
---

# Introduction



# What is SOAP?

SOAP its a protocol which uses XML for messages using HTTP (or any other protocols for transferring messages with each other with their specifics, were going talking about HTTP/HTTPS).

# What is WSDL?

WSDL is XML-based language used for describing the schema offered by a web service. It allowing clients to understand how to interact with services using SOAP, it defines the structure, operations, input/output messages, and communication protocols that a web service.

It's like a xml schema using to represents how to build your message to the server. Every time it seems like a swamp with an endless and scarieng contents. But if you would read it once - you will read wsdl of any difficulty. 

## How to find wsdl schema?

You need to have access to the api. For example - [http://www.dneonline.com/calculator.asmx](http://www.dneonline.com/calculator.asmx).

This is a SOAP api for testing or practising interaction using SOAP. They have a wsdl schema, to come in just update your link and add ?wsdl at the end ([http://www.dneonline.com/calculator.asmx?wsdl](http://www.dneonline.com/calculator.asmx?wsdl)) and you will get it!

# Secureness of SOAP

## Why it is Safe?

SOAP safe in first thanks to strictly standarts which requires to build xml only for rules described in schemes. You always known which data's you will receive in successfull and failed scenarios. You can also see that info in wsdl schema. Im sorry, but i dont want to explain wsdl schemes, its a content for another post :(

Consider a SOAP web service that interacts with a vault to retrieve API keys or authentication tokens securely.

## Example of http query which uses SOAP:

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

Our xml-soap-message placed in http-message body and we are also receiving similar http queries.

## Components of SOAP (what does it consists of)

### xml

SOAP message starts in describing which standard use to be. Some api's will requires specific version of xml, so you need be to sure about it.

```xml
<?xml version="1.0" encoding="utf-8"?>
```

### Envelope

Next, it must be an SOAP message which start with Envelope. There is a beginning element of every SOAP message. They define structures and namespaces used in the message. `xmlns:xsi` `xmlns:xsd` and `xmlns:soap` is a namespaces which used to standartize our message. Using documents from these links receiver of message can understand by which rules our message is builded.

```xml
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" # xmls
               xmlns:xsd="http://www.w3.org/2001/XMLSchema"
               xmlns:soap="http://schemasxmlsoap.org/soap/envelope/">
</soap:Envelope>
```

### Header

Header is an Optional element which contains metadata related to the message, such as authentication, session management, or routing information. In most messages it just may be empty. They have to contains namespaces, parameters and structures, similar to `body`.

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

Our soap-body contains the request to the specific resourse/method and response parameters.

- `GetInfo` - is a method or resource which we interacts in schema
- `param1` and `param 2` - is a our parameters which we sends to a server

```xml
<soap:Body>
  <GetInfo xmlns="http://example.com/">
    <param1>value1</param1>
    <param2>value2</param2>
  </GetInfo>
</soap:Body>
```

## Answer of SOAP message

Soap also returns xml document which also have `Envelope`, `Body`, but there you can reach the new element - `Fault`

### Fautl

Fault is an optional element that provides information about errors that occurred while processing messages.

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

WS-Security is a part that provides security mechanisms for web services. It ensures that the message has not been altered during transmission.

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

# How to debug and working with soap?

There is a lot of instruments to workings with SOAP, but i will recommend the **SoapUI**. There is ultimate tool which can help you automatically generate xml document and sends request inside it. It's just Postman from the xml world!

A lot of companies sending their soap apis with exported soapUI schemes. In one click you can import and receive all methods with all parameters, examples of requests/responses

# SOAP on Ruby

Many production projects preferring builds XML by himself, building a huge OOP monoliths which returns a string with XML document. This is a good way, but you might to think about supporting this huge piece of code. 

There is easiest way to sends requests.

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

But in production you will encounter difficult xml's which length is easy to achieving 100+ rows.

# Useful gems

Ruby have a few gems which will be good to solve problems:

## [Savon](https://github.com/savonrb/savon) - Heavy metal SOAP client which generates xml and sends requests on ruby

Rubies have a any of instruments for working with soap. The most popular is [savon](https://github.com/savonrb/savon)

```ruby
require 'savon'

# Initialize the client with the WSDL URL of the SOAP service
client = Savon.client(wsdl: 'http://www.dneonline.com/calculator.asmx?WSDL')

# Make a request to the 'Add' operation of the service
response = client.call(:add, message: { 'intA' => 5, 'intB' => 3 })
response.body
```

Gem using wsdl schema to build xml document by setuped rules. Actual generated xml will be:

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

There is ultimate gem which actual for now and will support any oldest versions of Ruby.

## [XMLDSig](https://github.com/benoist/xmldsig)

XMLDSig (XML Digital Signature) is a standard for signing XML documents to ensure their authenticity, integrity, and non-repudiation. The purpose of XMLDSig is to allow you to sign an XML document or a portion of it, providing proof that the document has not been altered since it was signed.

It is a hard-process to self signing your document, so there is gem [XMLDSig](https://github.com/benoist/xmldsig) which will make your life easier.

But there is bad news - gem is not currently supportind by their creators, but its works on newest ruby versions (works on my project with ruby 3.3.0)

# Conclusions

SOAP is a protocol which **strictly** requires to achieve specific rules. It's safe, it's incredibly compatible between different platforms thanks to a standarts. But it's harder than architectures styles (for example REST), they requires a reverent attitude to a building xml documents to being valid.
