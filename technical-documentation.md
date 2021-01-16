# Technical Documentation Message Alert (IBO)

!!! info "arc42"
    This documentation is based on the [arc42](https://sdos-central.gitlab-pages.kazan.atosworldline.com/sdos-central-docs/arc42) 
    template for documentation of software and system architecture.

    The documentation is written in [Markdown](https://sdos-central.
    gitlab-pages.kazan.atosworldline.com/sdos-central-docs/markdown).

## Introduction and Goals

The messaging system __Message Alert__ sends SMS or Email for certain events triggered by the authorization system (WLP-FO / frontoffice).

It manages subscriptions for such messages per card contract. Reading, creating and updating of subscription is done over services that Message Alert offers:

  - SOAP-service: used by the Online Banking Portal
  - REST-service: used by the card management system (IBO) - for the customer care

Configuration for Message Alert is stored in database tables, the configuration is divided into:

  - event type (advice code from WLP-FO) and card product (program of the card that triggered the event) specific configuration
  - card product specific configuration


__Documentation / Specification resources:__

  - Service Description: `S:\Daten\DT\4\04 Products - Services\Worldline™message alert\Leistungsbeschreibung Message Alert\Leistungsbeschreibung Message Alert 1.3 EN.pdf`
  - Specification documents: `S:\Daten\PROJEKTE\SMS\003 Message Alert - Copernicus`
  - Configuration documents (comdirect): `S:\Daten\PROJEKTE\SMS\400 SMS-Alert ComDirect\410 ComDir Konfiguration`
  - Specification WIKI (in German): [MediaWiki - Message Alert](https://defmcrvmis004/issuing/index.php?title=Produkt:Message_Alert)
  - Specification of the STNI interface from WLP-FO: [Sharepoint STNI](https://sp2013.myatos.net/organization/gbu/wl/ewl/coo/Acquiring%20products/AS-SharedCenter/Documents/Forms/AllItems.aspx?RootFolder=%2Forganization%2Fgbu%2Fwl%2Fewl%2Fcoo%2FAcquiring%20products%2FAS%2DSharedCenter%2FDocuments%2FDocumentation%2FSpecifications%2FInterfaces%2FSTNI)
  - Specification of the IBO services:
      - [Sharepoint IBO](https://sp2013.myatos.net/communities/cm1/SC-IBS/Shared%20Documents/IBO%20Reference)
      - [Sharepoint IBO Search Service](https://sp2013.myatos.net/communities/cm1/SC-IBS/Shared%20Documents/IBO%20Reference/Work%20in%20progress/IBO-EXTERNAL/WLP%20IBO-EXTERNAL%206.2%20-%20Service%20Contract%20v1.0.docx?Web=1)
      - [Sharepoint  IBO Add Address](https://sp2013.myatos.net/communities/cm1/SC-IBS/Shared%20Documents/IBO%20Reference/Work%20in%20progress/CUS/WLP%20CUS%2020Q1%20-%20Service%20Contract%20v6.1-DRAFT.docx?Web=1)
  - E2E Releasemanagement: [Confluence](https://confluence.worldline.com/display/GIES/Release+management)

<!--

### Requirements Overview

### Quality Goals

### Stakeholders


## Architecture Constraints

-->

## System Scope and Context

### Business Context

```plantuml format="svg"

rectangle events as "WLP-FO (STNI event)" {
  node OLW as "Online Watcher\n(OLW)"
  node WLPFO as "Worldline pay frontoffice\n(WLP-FO)"
  queue wlpfo_queue as "STNI-Event Queue"
}

rectangle subsription as "Subscriptions / Registration" {
  node IBO as "Card Management System\n(IBO)"
  interface ibo_soap as "SOAP"
  node PremiumGUI as "Premium GUI"
  node PremiumGateway as "Premium Gateway"
  node online_banking as "Online Banking\n(extern)" #LightGrey
  queue ebe_queue as "EBE Queue\n(external business events)"
}

rectangle message as "Message-Alert" {
  queue ma_snti_queue as "STNI-Event Queue"
  queue ma_ebe_queue as "EBE Queue"
  node message_alert as "Message-Alert" #Orange
  interface ma_json as "JSON"
  interface ma_soap as "SOAP"
  database ma_db as "Message-Alert" #Orange

  node reporting as "Reporting" #Chocolate
  node configuration as "Configuration-/\nTemplate-\nMaintenance" #Chocolate
}

rectangle provider as "Service Provider" {
  cloud sms as "SMS\n(Message Mobile, SAP)" #LightGrey
  cloud email as "E-mail" #LightGrey
}

rectangle output_channel as "Output Channel" {
  queue ing_queue as "ING-Queue" #LightGrey
  cloud ws_postbank as "Postbank" #LightGrey
  cloud nn_bank as "NN Bank" #LightGrey
}

OLW <--> WLPFO
WLPFO --> wlpfo_queue
wlpfo_queue --> ma_snti_queue : MQ-Series
ebe_queue --> ma_ebe_queue : MQ-Series
online_banking --> ma_soap
PremiumGUI --> PremiumGateway
PremiumGateway --> IBO
IBO --> ma_json
IBO <-- ibo_soap
IBO --> ebe_queue

ibo_soap <-- message_alert
ma_snti_queue --> message_alert
ma_ebe_queue --> message_alert
ma_json --> message_alert
ma_soap --> message_alert
message_alert <--> sms : SMPP
message_alert <--> email : SMTP

message_alert --> ma_db : JDBC

message_alert --> ing_queue : Active MQ
message_alert --> nn_bank : HTTP/JSON
message_alert --> ws_postbank : HTTP / JSON

reporting --> ma_db : stored procedures
configuration --> ma_db : stored procedures

```


| Module | Description |
| ------ | ----------- |
| WLP-FO | The authorization system (Worldline Pay Frontoffice)<br>WLP-FO sends events for transactions (STNI-requests), communication is done over a MQ-Series queue. |
| IBO | The card management system<br>The customer care service can look into and change the subscriptions and addresses for Message Alert. (This will be done over Premuim GUI / Gateway) |
| Online Banking | External bank portal offering the card holder functionalities to subscribe/unsubscribe for Message Alert. Also allows to change the address (Email or SMS that should be used). Changes over the Online Banking Portal will have to be propagated to IBO by Message Alert |
| Message Alert | The messaging system<br>Java Application running on JBoss EAP |
| Service Provider | SMS and Email provider that will deliver the alert messages to the card holder |
| Output Channel | Output channels for Message Service events |
| Configuration-/Template-Maintenance | Maintenace is done directly on the database holding the configuration data |
| Reporting | Reporting is done directly on the database |



<!--

### Technical Context


## Solution Strategy

-->


## Building Block View

### Whitebox Overall System

```plantuml format="svg"
node wlpfo as "WLP-FO"
node bank as "Online Banking" #LightGrey
node ibo as "IBO"
node provider as "Service Provider" #LightGrey
node output_channel as "Output Channel" #LightGrey

frame message_alert as "message-alert-ibo" #DarkOrange {
  component interfaces as "interfaces\n(Camel)" #Gold
  component services as "services\n(Camel)" #Gold
  component domain as "domain" #Gold
  component build as "build" #Gold
  database db as "DB" #Gold

  interfaces <-- build
  services <-- build
  domain <- build

  interfaces --> domain
  services --> domain

  interfaces --> services

  domain --> db : JDBC
}

wlpfo --> interfaces
ibo --> interfaces
bank --> interfaces
provider <--> interfaces
output_channel <--> interfaces
```


| Module | Description |
| ------ | ----------- |
| interfaces | Contains all modules that interact with external systems (incoming and outgoing requests). Interaction is done with Apache Camel as base framework. |
| services | Contains the modules defining different use cases of the messaging system (e. g. create subscription). The use cases are defined in Apache Camel routes. |
| domain | Contains the modules implementing the core business logic (e. g. the domain model) |
| build | Contains the build module that packages the application into a WAR file. Also contains the configuration files for JBoss for the Dev-Environment (e. g. the `standalone-ma.xml` and the init SQL script for the embedded database for DEV). |
| DB | The database storing the Message Alert domain data. Access is done via JDBC (Spring) |


### Whitebox Subscriptions

```plantuml format="svg"
node bank as "Online Banking" #LightGrey
node ibo as "IBO"

frame message_alert as "message-alert-ibo" #DarkOrange {
  package interfaces #Orange {
    component alert_subscription as "alert-subscription"
    component ibo_subscription as "ibo-subscription?"
    component ibo_services as "ibo-services?"
  }
  package services #Orange {
    component subscription_online as "subscription-online"
  }
  package domain #Orange {
    component model
    component subscriptions as "subscriptions"
    component event_processing as "event-processing" #Snow
  }
  database db as "ALERT_ACCOUNT\nALERT_ADDRESS"

  alert_subscription --> subscription_online : Camel Route
  ibo_subscription --> subscription_online : Camel Route
  model --> db : JDBC
  ibo_services <-- subscription_online : Camel Route

  ibo <-(0- ibo_services : SOAP

  subscription_online --> subscriptions : Bean
  subscriptions --> model

  subscription_online -> event_processing : Bean\n(Welcome SMS)
}

bank -0)-> alert_subscription : SOAP
ibo -0)-> ibo_subscription : JSON / REST
```


| Module | Description |
| ------ | ----------- |
| alert-subscription | Offers the SOAP services for the Online Banking Portal |
| ibo-subscription? | Offers the JSON / REST endpoint for IBO |
| ibo-services? |  |
| subscription-online |      |
| model |                |
| subscriptions |               |
| event-processing |               |
| ALERT_ACCOUNT<br>ALERT_ADDRESS |             |


### Whitebox Event Processing

```plantuml format="svg"
node wlpfo as "WLP-FO"
node IBO as "IBO"
node extern as "External Service" #LightGrey

cloud SAP #LightGrey
cloud mm as "Message Mobile" #LightGrey
cloud smtp as "Email" #LightGrey
cloud nn_bank as "NN Bank" #LightGrey
queue ing_bank as "ING Bank" #LightGrey

frame message_alert as "message-alert-ibo" #DarkOrange {
  package interfaces #Orange {
    package input {
      component wlpfo_switchout as "wlpfo-switchout"      
      component alert_event as "alert-event"
      component external_business_event as "external_business_event"
    }
    package output {
      component sms_provider_sap as "sms-provider-sap"
      component sms_provider_mm as "sms-provider-message-mobile"
      component smtp_provider as "smtp-provider"      
      component message_service_ntfy_ing as "message_service_ntfy_ing"      
      component message_service_ntfy_nn as "message_service_ntfy_nn"            
    }
  }
  package services #Orange {
    component alert_event_processing as "alert-event-processing"
    component message_service_processing as "message-service-processing"
    component smpp_delivery as "smpp-delivery"
    component subscription_online as "subscription-online" #Snow
  }
  package domain #Orange {
    component event_processing as "event-processing"
    component subscriptions as "subscriptions"
    component model
  }
  database db as "Message-Alert"

  model --> db : JDBC

  alert_event --> alert_event_processing
  wlpfo_switchout --> alert_event_processing
  wlpfo_switchout --> message_service_processing
  message_service_processing <--external_business_event
  sms_provider_sap <-- alert_event_processing
  sms_provider_mm <-- alert_event_processing
  smtp_provider <-- alert_event_processing
  alert_event_processing --> event_processing
  subscriptions --> model
  event_processing --> model

  sms_provider_sap --> smpp_delivery
  sms_provider_mm --> smpp_delivery
  smpp_delivery --> model
  extern --> alert_event : SOAP

  alert_event_processing <-- subscription_online : (Welcome SMS)

  smtp <-(0- smtp_provider
  SAP <-(0- sms_provider_sap
  mm <-(0- sms_provider_mm  
  ing_bank <-- message_service_ntfy_ing : ActiveMQ
  nn_bank <-(0- message_service_ntfy_nn : HTTP/JSON
}

wlpfo -0)-> wlpfo_switchout : MQ-Series
IBO -0)-> external_business_event : MQ-Series
```

| Module | Description |
| ------ | ----------- |
| wlpfo-switchout | Maven module responsible for STNI request reception and event creation.<br>Forwards to the next processing step based on the service type of the events. |
| external-business-event | Maven module responsible for handling external business events.<br>Forwards to the next processing step based on the service type of the events. Currently, all business events are forwarded to message event processing. |
| alert-event | Interface maven module offering a webservice for external systems that want to use Message Alert for SMS / email delivery |
| alert-event-processing | Maven module responsible for the event processing of Alert events (sending email and/or SMS to the cardholder) |
| message-service-processing | Maven module responsible for the event processing of Message Service events (sending events to the bank specific output channel, e. g. queue or webservice)
| event-processing | Domain maven module responsible for the core logic to create messages for Alert events |
| sms-provider-sap | Maven module responsible for sending SMS through SAP |
| sms-provider-message-mobile | Maven module responsible for sending SMS through Message Mobile |
| smtp-provider | Maven module responsible for sending email messages |
| smpp-delivery | Service maven module contains common logic for sending SMS (e. g. delivery receipt handling) |
| message_service_ntfy_ing | Service maven module that sends nofication to ing banks upon receiving events. |
| message_service_ntfy_nn | Service maven module that sends nofication to ing banks upon receiving STNI events from FO and business events from IBO. |
| subscription-online | Maven module handling the subscriptions to Message Alert, triggers a welcome SMS / email on subscription changes |

<!--

### Level 2

#### White Box *\<building block 1\>* {#_white_box_emphasis_building_block_1_emphasis}

#### White Box *\<building block 2\>* {#_white_box_emphasis_building_block_2_emphasis}

### Level 3

#### White Box \<\_building block x.1\_\> {#_white_box_building_block_x_1}

#### White Box \<\_building block x.2\_\> {#_white_box_building_block_x_2}

-->

## Runtime View

### Subscriptions Online Banking Portal

```plantuml format="svg"
autonumber

participant bank as "Online Banking\nPortal"
box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_ACCOUNT
  database ALERT_ADDRESS
  database ALERT_PACKET_CONFIG
end box
participant ibo as "IBO"

== Query Subscription ==
activate bank
bank -> ma : querySubscription
note left
Query subscriptions with PAN
end note
activate ma

ma -> ibo : search
note left
Search with PAN to retrieve
- Card Contract Reference
- Issuer Reference
- Product Reference
end note
activate ibo
ma <- ibo
deactivate ibo

ma -> ALERT_ACCOUNT : fetch account
note left
Select by CARD_CONTRACT_REF
end note
activate ALERT_ACCOUNT
ma <- ALERT_ACCOUNT
deactivate ALERT_ACCOUNT

alt Account exists
    ma -> ALERT_ADDRESS : fetch addresses
    note left
    Select by ALERT_ACCOUNT_ID
    end note
    activate ALERT_ADDRESS
    ma <- ALERT_ADDRESS
    deactivate ALERT_ADDRESS

    ma -> ALERT_PACKET_CONFIG : fetch packet
    activate ALERT_PACKET_CONFIG
    ma <- ALERT_PACKET_CONFIG
    deactivate ALERT_PACKET_CONFIG
else Account does NOT exist
    ma -> ALERT_ACCOUNT : insert account
    activate ALERT_ACCOUNT
    ma <- ALERT_ACCOUNT
    deactivate ALERT_ACCOUNT
end

bank <- ma : return subscriptions
note left
Return subscription with:
- ALERT_ACCOUNT_ID
- addresses
- packet
end note
deactivate ma

bank -> bank : display subscriptions
```

```plantuml format="svg"
autonumber

participant bank as "Online Banking\nPortal"
box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_ACCOUNT
  database ALERT_ADDRESS
  database ALERT_PACKET_CONFIG
end box
participant ibo as "IBO"

== Set Subscription ==
activate bank
bank -> ma : setSubscription
note left
Set subscriptions with:
- ALERT_ACCOUNT_ID
- addresses
- packet
end note
activate ma

ma -> ALERT_ACCOUNT : fetch account
note left
Select by ALERT_ACCOUNT_ID
end note
activate ALERT_ACCOUNT
ma <- ALERT_ACCOUNT
note right
Account with
- CARD_CONTRACT_REF
- ISSUER_REF
- PRODUCT_REF
must already exist
end note
deactivate ALERT_ACCOUNT

ma -> ALERT_ADDRESS : fetch addresses
note left
Select by ALERT_ACCOUNT_ID
end note
activate ALERT_ADDRESS
ma <- ALERT_ADDRESS
deactivate ALERT_ADDRESS

ma -> ALERT_PACKET_CONFIG : fetch packet
note left
Select by PACKET_NAME (bank-request)
and return ALERT_PACKET_ID
end note
activate ALERT_PACKET_CONFIG
ma <- ALERT_PACKET_CONFIG
deactivate ALERT_PACKET_CONFIG

ma -> ALERT_ACCOUNT : update account
activate ALERT_ACCOUNT
ma <- ALERT_ACCOUNT
deactivate ALERT_ACCOUNT

ma -> ALERT_ADDRESS : insert/update addresses
activate ALERT_ADDRESS
ma <- ALERT_ADDRESS
deactivate ALERT_ADDRESS

bank <- ma : return status code

bank -> bank : display result

group asynchronous
    ma -> ibo : update subscription
    note left
    TODO
    end note
    activate ibo
    ma <- ibo
    deactivate ibo
end
```

### Subscriptions IBO

```plantuml format="svg"
autonumber

participant ibo as "IBO"
box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_ACCOUNT
  database ALERT_ADDRESS
  database ALERT_PACKET_CONFIG
end box

== Set Subscription ==
activate ibo
ibo -> ma : setSubscription
note left
Set subscriptions with:
- CARD_CONTRACT_REF
- addresses
- packet
end note
activate ma

ma -> ALERT_ACCOUNT : fetch account
note left
Select by CARD_CONTRACT_REF
end note
activate ALERT_ACCOUNT
ma <- ALERT_ACCOUNT
deactivate ALERT_ACCOUNT

alt Account exists
    ma -> ALERT_ADDRESS : fetch addresses
    note left
    Select by ALERT_ACCOUNT_ID
    end note
    activate ALERT_ADDRESS
    ma <- ALERT_ADDRESS
    deactivate ALERT_ADDRESS
else Account does NOT exist
    ma -> ibo : search
    note left
    Search with Card Contract Reference
    to retrieve:
    - Issuer Reference
    - Product Reference
    end note
    activate ibo
    ma <- ibo
    deactivate ibo
end

ma -> ALERT_PACKET_CONFIG : fetch packet
note left
Select by PACKET_NAME (ibo request)
and return ALERT_PACKET_ID
end note
activate ALERT_PACKET_CONFIG
ma <- ALERT_PACKET_CONFIG
deactivate ALERT_PACKET_CONFIG

ma -> ALERT_ACCOUNT : insert/update account
activate ALERT_ACCOUNT
ma <- ALERT_ACCOUNT
deactivate ALERT_ACCOUNT

ma -> ALERT_ADDRESS : insert/update addresses
activate ALERT_ADDRESS
ma <- ALERT_ADDRESS
deactivate ALERT_ADDRESS


ibo <- ma : return status code
deactivate ma

ibo -> ibo : ...

```

### WLP-FO STNI Event Processing

#### STNI Switchout Processing

The STNI switchout processing is responsible for receiving STNI switchouts
from WLP-FO (over an MQ-Series queue). For each STNI switchout the events
for the issuer of this switchout are identified.

Events are identified in two ways:

- Their event criterias all match the incoming request (`EVENT_TEMPLATE`, `EVENT_CRITERIA`)
- The incoming request contains Online Watcher advice code that match (`ALERT_OLWADVPRIO`)

For each identified event an entry is written to `ALERT_EVENT`.

The identified events are split by service type and subscription type.
For each service type / subscription type combination an event container
holding the events is created.

Before forwarding the event container to the next processing step, Message
Alert checks if a subscription for this service type (and subscription type)
exists.

!!! Note
    The `ALERT_EVENT` is written after identifying the event from the issuer
    configuration and before the check if a subscription exists.

    This will result in a lot of entries in `ALERT_EVENT` for events that do
    not result in a message. But it will allow to identify easily if an event
    was identified and if a wrong subscription was the cause for a missing
    message.

!!! Info
    Message Alert is capable of reading from multiple STNI MQ-Series Queues.
    Connecting different WLP-FO instances is therefor possible.

    Message Alert is not able to identify from which WLP-FO instance a STNI
    request originated, thus the WLP-FO instances have to send the same
    STNI request structure (WLP-FO head release) and have to include the
    necessary information (e. g. issuer reference).

```plantuml format="svg"
autonumber

queue WLPFO as "WLP-FO (STNI)"
box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_COMPANY as "ALERT\nCOMPANY"
  database EVENT_CRITERIA as "EVENT\nTEMPLATE\n+ CRITERIA"
  database ALERT_OLWADVPRIO as "ALERT\nOLWADV\nPRIO"
  database ALERT_EVENT as "ALERT\nEVENT"
  database SUBSCRIPTIONS as "SUBSCRIP\nTIONS\n+ SERVICE"
  queue ma_event as "Alert\nEvent\nQueue"
  queue ma_message_service as "Message\nService\nQueue"
end box

== STNI Request Processing ==

activate WLPFO
WLPFO -> ma : Read STNI request\nfrom queue
note left
STNI request with:
- Issuer ref
- Card contract ref
- OLW advice codes
- Fields to identify
  PIN-change and
  instalment credit
  (e. g. BusinessCaseTyp,
  Declined,
  TransactionType_2)
end note
deactivate WLPFO

activate ma
ma -> ALERT_COMPANY : Fetch isuer configuration
activate ALERT_COMPANY
return

ma -> EVENT_CRITERIA : Fetch criterias for issuer\nand ref-system
activate EVENT_CRITERIA
return
deactivate EVENT_CRITERIA
loop for each event in EVENT_TEMPLATES with REF_SYSTEM = "FO"
  ma -> ma : Check if event criterias match
  note left
    If all event criterias for an event match the
    STNI request, this new event will be created
  end note
  ma -> ma : Create new event
  ma -> ma : Add event to event list
end

alt STNI contains OLW advice codes
  ma -> ALERT_OLWADVPRIO : Find relevant OLW advice codes
  note left
    If the STNI request contains
    OLW advice codes, then select
    advice codes with highest
    priority per group
  end note
  activate ALERT_OLWADVPRIO
  return
  deactivate ALERT_OLWADVPRIO

  loop for each OLW advice code with\nhighest prio per group)
    ma -> ma : Create new event for advice code
    ma -> ma : Add event to event list
  end
end

loop for each event in event list
  ma -> ALERT_EVENT : Store event
  activate ALERT_EVENT
  deactivate ALERT_EVENT
end

ma -> ma : Split events into Alert events\nand Message Service events
note left
  Create a new event container for Alert events
  and Message Service events
end note
activate ma

ma -> SUBSCRIPTIONS : Fetch subscription + service
note left
  Find subscription and subscription service
  for the service type and subscription type
  in the event container.
end note
activate SUBSCRIPTIONS
return

ma -> ma : Filter events without subscription

alt Subcription for service found
  ma -> ma_event : send alert event container\nto event processing
  note left
    List of events for the same STNI request
    are forwarded to the event processing
    (have to be processed together as one event
    can have an impact on the other)
  end note
  activate ma_event
  ma -> ma_message_service : send message service container\nto event processing
  deactivate ma
  deactivate ma
  activate ma_message_service
end

== Event Processing ==
```

#### Alert Event Processing

- Fetch `ALERT_MASTERDATA` to identify alert service type
- Check authorization result and value (this might change the advice code)
- Fetch email and msisdn addresses (from `ALERT_ADDRESS`) or validate an
  address is given in the incoming event (for address source = `EVNT`)

```plantuml format="svg"
autonumber

box "Message-Alert"
  queue ma_event as "Internal\nEvent\nQueue"
  participant ma as "Message-Alert"
  database ALERT_MASTERDATA as "ALERT\nMASTERDATA"
  database ALERT_ADDRESS as "ALERT\nADDRESS"
end box

activate ma_event
...

== Alert Event Processing ==
note left of ma
  Event Container with events:
  - ADVICE_CODE / EVENT_NAME
  - CARD_CONTRACT_REFERENCE
  - ISSUER_REFERENCE
  - service type
  - subscription type

  (all events have the same issuer
  and card contract reference;
  events may belong to different
  service types)
end note

ma_event -> ma
deactivate ma_event
activate ma

loop for each event in the container
  ma -> ALERT_MASTERDATA : Fetch masterdata
  note left
    Select by
    - PRODUCT_REF (account)
    - ADVICE_CODE (event)

    Returns masterdata with:
    - ADDRESS_SOURCE
    - SERVICE_TYPE
    - REQUEST_TYPE
    - ALERT_TEMPLATE_ID
  end note
  activate ALERT_MASTERDATA
  return

  alt If CHECK_AUTHRESULT 'Y'
    ma -> ma : Check auth result/value (see diagram below)
    activate ma
    return May return a new advice code\nfor the message template
  end

  alt If ADDRESS_SOURCE is not "EVNT"
    ma -> ALERT_ADDRESS : Fetch addresses
    note left
      Select by
      - ADDRESS_SOURCE (masterdata)
      - SERVICE_TYPE (masterdata)

      Returns addresses with:
      - msisdn
      - email
    end note
    activate ALERT_ADDRESS
    return
    note left
      If no address is found, stop processing
      this event (no subscription)
    end note
  else ADDRESS_SOURCE = "EVNT"
    ma -> ma : Check if event contains address information
  end

  == Message Creation and Delivery ==
  ma -> ma : ...
end
```

#### Auth Result and Auth Value check

!!! Info
    For the template the advice code from the auth result/value
    check will be used (identifies the entry in `ALERT_EVENT_MAP`
    that holds the reference to `ALERT_TEMPLATES`).

    This also means that not every advice code in `ALERT_EVENT_MAP`
    has an entry in `ALERT_MASTERDATA`. The advice codes from the    auth result/value check can only be found in `ALERT_EVENT_MAP`.

```plantuml format="svg"
autonumber

box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_MASTERDATA as "ALERT\nMASTERDATA"
  database ALERT_PACKET_RULE as "ALERT\nPACKET\nRULE"
  database AUTH_RESULT_LIST as "AUTH\nRESULT\nLIST"
end box

activate ma

== Auth Result / Value Check ==

ma -> ALERT_PACKET_RULE : Fetch packet rule
note left
  Select by
  - ALERT_PACKET_ID (account)
  - ADVICE_CODE (event)

  Returns rule with:
  - AUTH_RESULT_GROUP
  - AUTH_VALUE_DEF
end note
activate ALERT_PACKET_RULE
return
note left
  - If no rule found, then stop processing
  - If rule found but no group defined,
    continue with advice code from event
  - If rule found with group defined, fetch
    auth result list
end note

ma -> AUTH_RESULT_LIST : Fetch new advice code
note left
  Select by:
  - AUTH_RESULT_GROUP (packet rule)
  - AUTH_RESULT (event - substring match)

  Returns auth result with:
  - MATCH_ADVICE_CODE (new advice code)
end note
activate AUTH_RESULT_LIST
return
note left
  - If not found, stop processing
  - If found but MATCH_ADVICE_CODE empty,
    process with advice code from event
  - If found and MATCH_ADVICE_CODE set,
    process with MATCH_ADVICE_CODE as
    new ADVICE_CODE
end note

group Only if MATCH_ADVICE_CODE is not empty
  ma -> ALERT_PACKET_RULE : Fetch packet rule for new advice code
  activate ALERT_PACKET_RULE
  return
end

ma -> ma : Check auth value against\n AUTH_VALUE_DEF
note left
  If auth value from event is lower
  than AUTH_VALUE_DEF from
  the packet rule stop processing
end note

ma -> ma : Set ALERT_EVENT_MAP advice code
note left
  If MATCH_ADVICE_CODE is not empty,
  it is the advice code for the event map
  configuration (identifies the template).

  Otherwise the original advice code from
  will be used.
end note
```

#### Alert Message Creation and Delivery

- Fetch configuration for the message creation
- Create messages
- Check country / area code for msisdn
- Forward messages to service provider

```plantuml format="svg"
autonumber

box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_TEMPLATES as "ALERT\nEVENT_MAP\n+ TEMPLATES"
  database ALERT_CONFIGURATION as "ALERT\nCONFIGURATION\n+ PARAMETER"
  database ALERT_COUNTRY_CODE as "ALERT\nCOUNTRY_CODE\n+ AREA_CODE"
  database MESSAGE_LOG as "MESSAGE\nLOG"
  queue ma_message as "Internal\nMessage\nQueue"
end box

== Message Creation ==
activate ma
note left of ma
  Event with:
  - ADVICE_CODE
  - CARD_CONTRACT_REF
  - ISSUER_REF

  Masterdata with:
  - SERVICE_TYPE
  - REQUEST_TYPE
  - PROCESSING_TYPE

  Address with:
  - msisdn or email
end note

ma -> ALERT_TEMPLATES : Fetch templates
note left
  Select by:
  - ALERT_TEMPLATE_ID
  - Alert service type, etc.

  Returns:
  - Html/Text template strings
end note
activate ALERT_TEMPLATES
return

ma -> ALERT_CONFIGURATION : Fetch configuration
note left
  Select by:
  - PRODUCT_REF
  - SERVICE_TYPE

  Returns:
  - COMPANY_ID
  - Template variables
end note
activate ALERT_CONFIGURATION
return

group Only if CHECK_COUNTRYCODE 'Y'
ma -> ALERT_COUNTRY_CODE : Check country code
note left
  TODO
end note
activate ALERT_COUNTRY_CODE
ma <- ALERT_COUNTRY_CODE
deactivate ALERT_COUNTRY_CODE
end group


ma -> ma : Create Message
note left
  Create message with template
  variables from configuration,
  masterdata, event
end note

ma -> MESSAGE_LOG : Store log entry
activate MESSAGE_LOG
return


ma -> ma_message
deactivate ma
activate ma_message

== Message Delivery ==

```

```plantuml format="svg"
autonumber

box "Message-Alert"
  queue ma_message as "Internal\nMessage\nQueue"
  participant ma as "Message-Alert"
  database MESSAGE_LOG as "MESSAGE\nLOG"
end box
participant sms_email as "SMS/e-mail"

== Message Delivery ==

activate ma_message

ma_message -> ma
deactivate ma_message
activate ma

ma -> sms_email
activate sms_email
deactivate ma

ma <- sms_email : Delivery receipt
deactivate sms_email
activate ma
ma -> MESSAGE_LOG : Update log entry
activate MESSAGE_LOG
return

```

#### Message Service Event Processing

- Check `SUBSCRIPTION_EVENT`
- Fetch `EVENT_DATA_MAPPING`
- Send message to output channel
- Create `MESSAGE_LOG` entry

```plantuml format="svg"
autonumber

box "Message-Alert"
  queue ma_event as "Internal\nEvent\nQueue"
  participant ma as "Message-Alert"
  database SUBSCRIPTION_EVENT as "SUBSCRIPTION\nEVENT"
  database EVENT_DATA_MAPPING as "EVENT\nDATA\nMAPPING"
  database MESSAGE_LOG as "MESSAGE\nLOG"
end box
box "Output Channel" #White
  participant ING_QUEUE as "ING_QUEUE\nor\nNN_BANK (external)" 
end box

activate ma_event
...

== Message Service Event Processing ==
note left of ma
  Event Container with events:
  - ADVICE_CODE / EVENT_NAME
  - CARD_CONTRACT_REFERENCE
  - ISSUER_REFERENCE
  - service type
  - subscription type

  (all events have the same issuer
  and card contract reference;
  events may belong to different
  service types)
end note

ma_event -> ma
deactivate ma_event
activate ma

loop for each event in the container
  ma -> SUBSCRIPTION_EVENT : Fetch event subscriptions
  note left
    Select by
    - EVENT_NAME
  end note
  activate SUBSCRIPTION_EVENT
  ma <- SUBSCRIPTION_EVENT
  note left
    In generic part 2 an additional
    check for the subscription parameters
    will be necessary (including reevaluating
    if the event should be sent)
  end note
  deactivate SUBSCRIPTION_EVENT

  ma -> ma : Forward to output channel
  activate ma

  == Message Creation and Delivery ==
  ma -> EVENT_DATA_MAPPING : Fetch event data mapping
  activate EVENT_DATA_MAPPING
  return

  ma -> ma : Create outgoing event
  ma -> ING_QUEUE : Send event to output channel
  activate ING_QUEUE

  ma -> ma : Create message result
  deactivate ma

  ma -> MESSAGE_LOG : Store message log entry
  activate MESSAGE_LOG
  return
  deactivate ma
end
```
### IBO Business Event Processing

#### IBO Business Event Processing

IBO Business Event Processing module is responsible for receiving Business Events (EBE)
from IBO (over an MQ-Series queue). For each EBE switchout the events
for the issuer of this switchout are identified.

Events are identified in two ways:

- Their event criterias all match the incoming request (`EVENT_TEMPLATE`, `EVENT_CRITERIA`)
- The incoming request contains Online Watcher advice code that match (`ALERT_OLWADVPRIO`)

For each identified event an entry is written to `ALERT_EVENT`.

The identified events are split by service type and subscription type.
For each service type / subscription type combination an event container
holding the events is created.

Before forwarding the event container to the next processing step, Message
Alert checks if a subscription for this service type (and subscription type)
exists.

!!! Note
    The `ALERT_EVENT` is written after identifying the event from the issuer
    configuration and before the check if a subscription exists.

    This will result in a lot of entries in `ALERT_EVENT` for events that do
    not result in a message. But it will allow to identify easily if an event
    was identified and if a wrong subscription was the cause for a missing
    message.

!!! Info
    Message Alert is capable of reading from multiple STNI MQ-Series Queues.
    Connecting different WLP-FO instances is therefor possible.

    Message Alert is not able to identify from which WLP-FO instance a STNI
    request originated, thus the WLP-FO instances have to send the same
    STNI request structure (WLP-FO head release) and have to include the
    necessary information (e. g. issuer reference).

```plantuml format="svg"
autonumber

queue IBO as "IBO (EBE)"
box "Message-Alert"
  participant ma as "Message-Alert"
  database ALERT_COMPANY as "ALERT\nCOMPANY"
  database EVENT_CRITERIA as "EVENT\nTEMPLATE\n+ CRITERIA"
  database ALERT_OLWADVPRIO as "ALERT\nOLWADV\nPRIO"
  database ALERT_EVENT as "ALERT\nEVENT"
  database SUBSCRIPTIONS as "SUBSCRIP\nTIONS\n+ SERVICE"
  queue ma_event as "Alert\nEvent\nQueue"
  queue ma_message_service as "Message\nService\nQueue"
end box

== IBO EBE Request Processing ==

activate IBO
IBO -> ma : Read STNI request\nfrom queue
note left
EBE request with:
- Issuer ref
- Event kind
- Card id
- Pan (current & previous)
- Blocking status
end note
deactivate IBO

activate ma
ma -> ALERT_COMPANY : Fetch isuer configuration
activate ALERT_COMPANY
return

ma -> EVENT_CRITERIA : Fetch criterias for issuer\nand ref-system
activate EVENT_CRITERIA
return
deactivate EVENT_CRITERIA
loop for each event in EVENT_TEMPLATES with REF_SYSTEM = "IBO"
  ma -> ma : Check if event criterias match
  note left
    If all event criterias for an event match the
    EBE request, this new event will be created
  end note
  ma -> ma : Create new event
  ma -> ma : Add event to event list
end

loop for each event in event list
  ma -> ALERT_EVENT : Store event
  activate ALERT_EVENT
  deactivate ALERT_EVENT
end

ma -> ma : Split events into Alert events\nand Message Service events
note left
  Create a new event container for Alert events
  and Message Service events
end note
activate ma

ma -> SUBSCRIPTIONS : Fetch subscription + service
note left
  Find subscription and subscription service
  for the service type and subscription type
  in the event container.
end note
activate SUBSCRIPTIONS
return

ma -> ma : Filter events without subscription

alt Subcription for service found
  ma -> ma_event : send alert event container\nto event processing
  note left
    List of events for the same STNI request
    are forwarded to the event processing
    (have to be processed together as one event
    can have an impact on the other)
  end note
  activate ma_event
  ma -> ma_message_service : send message service container\nto event processing
  deactivate ma
  deactivate ma
  activate ma_message_service
end

== Event Processing ==
```

#### Message Service Event Processing

- Check `SUBSCRIPTION_EVENT`
- Fetch `EVENT_DATA_MAPPING`
- Send message to output channel
- Create `MESSAGE_LOG` entry

```plantuml format="svg"
autonumber

box "Message-Alert"
  queue ma_event as "Internal\nEvent\nQueue"
  participant ma as "Message-Alert"
  database SUBSCRIPTION_EVENT as "SUBSCRIPTION\nEVENT"
  database EVENT_DATA_MAPPING as "EVENT\nDATA\nMAPPING"
  database MESSAGE_LOG as "MESSAGE\nLOG"
end box
box "Output Channel" #White
  participant NN_BANK as "NN_BANK" #LightGray
end box

activate ma_event
...

== Message Service Event Processing ==
note left of ma
  Event Container with events:
  - ADVICE_CODE / EVENT_NAME
  - CARD_CONTRACT_REFERENCE
  - ISSUER_REFERENCE
  - service type
  - subscription type

  (all events have the same issuer
  and card contract reference;
  events may belong to different
  service types)
end note

ma_event -> ma
deactivate ma_event
activate ma

loop for each event in the container
  ma -> SUBSCRIPTION_EVENT : Fetch event subscriptions
  note left
    Select by
    - EVENT_NAME
  end note
  activate SUBSCRIPTION_EVENT
  ma <- SUBSCRIPTION_EVENT
  note left
    In generic part 2 an additional
    check for the subscription parameters
    will be necessary (including reevaluating
    if the event should be sent)
  end note
  deactivate SUBSCRIPTION_EVENT

  ma -> ma : Forward to output channel
  activate ma

  == Message Creation and Delivery ==
  ma -> EVENT_DATA_MAPPING : Fetch event data mapping
  activate EVENT_DATA_MAPPING
  return

  ma -> ma : Create outgoing event
  ma -> NN_BANK : Send event to output channel
  activate NN_BANK

  ma -> ma : Create message result
  deactivate ma

  ma -> MESSAGE_LOG : Store message log entry
  activate MESSAGE_LOG
  return
  deactivate ma
end
```

<!--

## Deployment View

### Infrastructure Level 1

### Infrastructure Level 2

#### *\<Infrastructure Element 1\>* {#__emphasis_infrastructure_element_1_emphasis}

-->

## Cross-cutting Concepts

### Data model / Persistence

Database table description is documented in [Data Model](data-model.md)

#### Subscriptions (derived from Account and Address)

The data model used for the subscription REST API (for IBO) and subscription
Soap API (for the Online Banking Portal). It is mapped into and from the
Account and Address database tables.

```plantuml format="svg"

class SubscriptionDto
SubscriptionDto : AlertAccountId
SubscriptionDto : CardContractRef
SubscriptionDto : ProductRef
SubscriptionDto : IssuerRef
SubscriptionDto : AlertPacket

class ServiceDto
ServiceDto : AddressSource

class ContactDto
ContactDto : AddressType
ContactDto : Value

SubscriptionDto "1" --> "*" ServiceDto
ServiceDto "1" --> "0..2" ContactDto

```

| Table           | Description |
| --------------- | ----------- |
| SubscriptionDto | One entry per registered card contract |
| ServiceDto      | List of registered services (ALRT, INST, NTFY) |
| ContactDto      | List of registered addresses for a service (MSISDN, EMAIL) |

#### IBO data model

```plantuml format="svg"

class SearchOutput

class CardInfo
CardInfo : issuerRef
CardInfo : cardNumber
CardInfo : cardContractRef

class PersonInfo
class AddressInfo
class AddressUsageInfo

class ExtContractSubscription
ExtContractSubscription : productRef


SearchOutput --> "*" CardInfo
SearchOutput --> "*" PersonInfo
SearchOutput --> "*" ExtContractSubscription

PersonInfo --> "*" AddressInfo
AddressInfo --> "*" AddressUsageInfo

```

#### Event / Message data

```plantuml format="svg"
package IBO as "IBO-EBE" {
  class ExternalBusinessEventDto
}
package WLPFO as "WLP-FO" {
  class STNIRequest
  class Generic
  class Notification
  class OnlineWatcher
  class Account

  STNIRequest --> "1" Generic
  STNIRequest --> "1" Notification
  STNIRequest --> "1" OnlineWatcher
  STNIRequest --> "*" Account
}

package ma as "Message Alert Events" {
  class EventContainerDto
  EventContainerDto : alertCompanyId
  EventContainerDto : issuerRef
  EventContainerDto : productRef
  EventContainerDto : cardContractRef
  EventContainerDto : cardNumber
  EventContainerDto : cardNumberRange

  class EventDto
  EventDto : keyValues
  EventDto : adviceCode
  EventDto : actionCode
  EventDto : eventName
  EventDto : eventLogId
  EventDto : authValue
  EventDto : authResult
  EventDto : serviceType
  EventDto : subscriptionType
  EventDto : subscriptionId
  EventDto : subscriptionReference

  STNIRequest ..> "*" EventContainerDto : transformed into n-events\ngrouped by Alert and Message Service events
  ExternalBusinessEventDto ..> "*" EventContainerDto : transformed into n-events\ngrouped by Alert and Message Service events
  EventContainerDto --> "*" EventDto

  object ALERT_EVENT
  ALERT_EVENT : EVENT_ID (PK)
  ALERT_EVENT : CREATE_DATE
  ALERT_EVENT : CARD_CONTRACT_REF
  ALERT_EVENT : PRODUCT_REF
  ALERT_EVENT : ISSUER_REF
  ALERT_EVENT : RULE_NAME / ADVICE_CODE
  ALERT_EVENT : TXN_REFERENCE
  EventDto ..> ALERT_EVENT
}


package ma_messages as "Message Alert Messages" {
  class MessageDto

  object MESSAGE_LOG
  MESSAGE_LOG : LOG_ID (PK)

  EventDto ..> "0..2" MessageDto
  MessageDto ..> MESSAGE_LOG
}
```

| Table | Description |
| ------ | ----------- |
| ALERT_EVENT | Logs incoming events (e. g. STNI requests from WLP-FO) |
| MESSAGE_LOG | Logs status of outgoing messages (email or sms) |

### Code Conventions and Recommendations

#### General recommendations

  - Make use of Lombok annotations for data transfer objects (DTO): 

    ```java
    @Data
    @EqualsAndHashCode
    @ToString
    public class MyDataClassDto {
        private BigDecimal myField;
        private String myOtherField;
    }
    ```

    !!! info
        Lombok is already defined as a maven dependency in the
        `message-alert-ibo` parent `pom.xml`

  - Use Lomboks builder annotation to use fluent API for object creation
    (e. g. in unit tests):

    ```java
    @Data
    @EqualsAndHashCode
    @ToString
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public class EventDto {

        private String cardContractRef;
        private String adviceCode;

        // ...
    }
    ```

    ```java
        @Test
        public void withSuccessProcessingInsertsEventAndMessageLog() throws Exception {
            EventDto eventDto = EventDto.builder()
                .cardContractRef(ANY_CARD)
                .adviceCode(ANY_ADVICE_CODE)
                .build();

            // ...
        }
    ```

  - Prefer Lombok for logging with `@Slf4j` annotation:

    ```java
    @Slf4j
    public class MyClass {
        public void myMethod() {
            log.warn("message text");
        }
    }
    ```

  - Make use of constructor injection for required dependencies and only use setter injection for optional dependencies:

    ```java
    @Component
    public class MySpringBean {
        private final OtherRequiredBean otherBean;

        @Autowired
        public MySpringBean(OtherRequiredBean otherBean) {
            Assert.notNull(otherBean, "otherBean must be specified");

            this.otherBean = otherBean;
        }
    }
    ```

  - Inject configuration values from `wlsi-messageAlertIBO.properties` in bean constructor as well:

    ```java
    @Component
    public class MySpringBean {
        private final String toEndpoint;

        public MySpringBean(@Value("${toEndpoint}") final String toEndpoint) {
            Assert.notNull(toEndpoint, "toEndpoint must be specified in properties file (toEndpoint)");

            this.toEndpoint = toEndpoint;
        }
    }
    ```

  - Assert that required dependencies are available in the constructor (see example code above)

#### Naming Conventions

TODO

#### Apache Camel

  - Prefer Java 8 lambda expressions (e. g. `process(MyProcessor::myMethod)`) over String definitions (e. g. `bean(MyBean.class, "myMethod")`)

  - Prefer lambda expressions over subclassing Camel classes like `Processor`

  - Prefer converting objects with `transform().body(MyDataDto.class, MyConverter::convertMethod)` instead of using `TypeConverter`

__Examples__

```java
@Component
class MyRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        from(ENDPOINT_ID).routeId(ROUTE_ID)
            .process(this::process);
    }

    private void process(Exchange exchange) {
        // do something with the exchange
        // Note: you have access to the exchange headers as well as the in-coming and out-going message
        // Note: you can also throw exception
    }
}
```

```java
@Component
class MyRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        from(ENDPOINT_ID).routeId(ROUTE_ID)
            .process
                .body(MyDataClassDto.class, this::process);
    }

    private void process(MyDataClassDto body) {
        // do something with the body
        // Note: you only have access to the body
        // Note: it is not possible to throw checked exceptions!
    }
}
```

```java
@Component
class MyRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        from(ENDPOINT_ID).routeId(ROUTE_ID)
            .transform
                .body(MyDataClassDto.class, this::transform);
    }

    private MyNewDataClassDto transform(MyDataClassDto body) {
        // do something with the body and return the new body for the exchange
        return new MyNewDataClassDto();
    }
}
```


#### JUnit-Tests

  - Structure JUnit test methods into __arrange__ (test setup), __act__ (execution of the method under test) and __assert__ (validate execution result):

    ```java
    @Test
    public void withEmptyRequestReturnsTrue() {
        // arrange
        STNIRequest request = new STNIRequest();
        STNIRequestChecker underTest = new STNIRequestChecker();

        // act
        boolean actual = underTest.containsNonRelevantAdviceCodes(request);

        // assert
        assertTrue(actual);
    }
    ```

  - Name pattern for test methods: __with__ Precondition __Returns__ ExpectedResult
  - Name the object you want to test as `underTest`
  - Make use of Hamcrest for validation/assertions (e. g. use `assertThat(actual, is("expected")))` instead of `assertEquals(acutal, "expected")`

    ```java
    @Test
    public void withPreconditionReturnsExpectedResult() {
        // arrange
        ...

        // act
        String actual = underTest.methodToTest(obj);

        // assert
        // prefer:
        assertThat(actual, is("expected")));

        // over:
        assertEquals(acutal, "expected");
    }
    ```

  - Focus on unit tests __not__ integration tests
  - Avoid initializing Spring Contexts in JUnit tests
  - Use `Mockito` for mocking objects

!!! tip
    To get Code Assist in Eclipse for static imports, add the classes to the Favorites list in the Eclipse preference under `Java > Editor > Content Assist > Favorites`

## Design Decisions

__Should queues be used to implement retry mechanism if a external resource is not available?__

  - __Pro:__

      - Already used in message-alert for sempris

  - __Contra:__

      - Order in which retry-messages are processed is hard to garantuee (e. g. might lead to wrong subscriptions)

  - __Alternative:__ Store status and timestamp in database and have a cron job that retries failed entries.

  - __Decision:__

      - Queues will be used
      - Order is guaranteed by using queue transactions (a message is only remove after being successfully processed)

__Separate DB schema for each card management system?__

  - __Pro:__

      - Separation from "old" Infoserver database schema/tables/procedures
      - Lower maintenance downtime if Infoserver is not used as database server

  - __Contra:__

      - May be more implementation effort

  - __Decision:__

      - A new DB schema will be used for Message Alert for IBO that will (in production) not be on the Infoserver


<!--


## Quality Requirements

### Quality Tree

### Quality Scenarios


## Risks and Technical Debts


## Glossary
-->

