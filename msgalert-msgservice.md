# Integration of Message Service into Message Alert

## Requirements for Message Service IBO

Migration of Message Service for IBO as part of the Copernicus project is devided into two parts:

 1. The first part migrates the functionality required for NN and OP (for Card Control).
 2. The second part migrates the functionality required for Postbank and ING (both banks are set up on Message Service for Sempris application)

### Part 1 (NN, OP - Card Control)

__Requirements for Card Control:__

  - Only events source are STNI-switchouts from WLP-FO (no CMS-events)
      - Version of WLP-FO will be the "head release" (STNI Specification 1.12)
      - Communcation channel for the STNI-switchouts are multiple Queues
  - No dependencies to a card management system (CMS)
      - No dependency to IBO/GODS, Sempris/Infoserver
      - Outgoing events only hold fields from STNI-switchout or from the static configuration
  - Only static subscriptions for card programs (card number ranges)
      - Subscription is part of the configuration
      - No subscription web services required

__Non functional requirements:__

  - Reduce dependency on specific database management system (DBMS), namely Oracle DB
  - The database server should have minimal maintanace downtimes
      - Infoserver DB should not be used


### Part 2 (Postbank, ING)

__Requirements for Postbank and ING:__

  - CMS as event source (IBO/GODS)
  - Subscriptions for cards and accounts (Postbank)
      - Web services for managing subscriptions
      - Subscriptions can override event criterias


## Solution Stragies

### I. New Application for Message Service IBO

__Advantages:__

  - No impact for other customers
      - No regression tests required
  - Moving to a different DB-server is easy
  - Moving away from stored procedures has no risks of breaking existing functionality

__Disadvantages:__

  - Another "Message" application (in total four)
      - Increased maintenance costs
      - Set up for DEV, QA, Production has to be done "from scratch"
      - Difficult to consolidate into one application afterwards

### II. Integrate into Message Service for Sempris

__Advantages:__

  - Only one Message Service application (in total there will be three "Message" applications)
      - After all customers have migrated to IBO there will be two "Message" applications

__Disadvantages:__

  - High risk of breaking existing functionality
      - Current implementation is highly coupled with Infoserver
  - High implementation costs
      - Current implementation is highly coupled with Infoserver
  - Difficult to move away from Infoserver DB
  - Regression tests for existing functionality and customers required (Postbank, ING)

### III. Integrate into Message Alert for IBO

__Advantages:__

  - Only one "Message" application for IBO (in total there will be three "Message" applications)
      - After all customers have migrated to IBO there will be only one "Message" application
  - Moving to a different DB-server is already done
  - Common functionalities between Message Service and Message Alert are only implemented once
      - CMS-events
      - Event criteria checks
      - Shared DB tables
  - No extra set up costs for DEV, QA, Prod
      - Environments are already available
      - Deployment process is already established
      - DB-schema already exists

__Disadvantages:__

  - Integration of the Message Service functionality may lead to a more complex application
      - Additional cost for analysis and specification
  - Regressiontests for Message Alert IBO is necessary (comdirect)
  - Possible migration of data for existing Message Alert customers necessary (comdirect)

### Preliminary Decision

The most promising solution is __III. Integrate into Message Alert for IBO__. This will be analyzed in detail in the following chapters.


## Integration into Message Alert for IBO

### Configuration

!!! abstract
    __Message Alert__:
    
      - Events assigned to service types (alert, instalment, notification)
      - __Main configuration of events is done on card program level__
      - Event filtering and event identification is done on issuer level
      - _Card subscriptions for service types and packets (the later only applies for service type "alert")_ (this will be discussed in the "Subscriptions" chapter)

    __Message Service__:
    
      - __Event configuration is done on issuer level__
      - Event filtering and event identification is done on issuer level
      - _Card/account/program subscriptions for events_ (this will be discussed in the "Subscriptions" chapter)

    __Integration__:

      - Events assigned to service types (alert, instalment, notification)
          - New service type for "Message Service" events
      - __Two options for event configuration__:
         1. Event configuration on card program level
              - Message Service configuration from `EVENT_TEMPLATE` has to be duplicated for each card program in `ALERT_MASTER_DATA`
         2. Event configuration split into issuer and card program configuration
               - Too complex?
         3. Event configuration on issuer level
              - Message Alert configuration from `ALERT_MASTER_DATA` has to be consolidated for each issuer in `EVENT_TEMPLATE`
              - Maybe a configuration which events are "active" for a card program is necessary
      - Event filtering and event identification is done on issuer level
          - New `EVENT_TEMPLATE` table for issuer level configuration of events (e. g. event source)

!!! question
    __Q1__

    :   Is it possible to define most configuration in `ALERT_MASTER_DATA` on issuer level? Defining `ADDRESS_SOURCE`, `SERVICE_TYPE` on issuer level allows to check for existing subscriptions early in the processing chain.

#### Database Tables

Issuer Configuration

```plantuml format="svg"

object EVENT_CRITERIA {
    ISSUER_REF
    ADVICE_CODE
    CRITERIA_ENTRY
    RULE_OPERATOR
    RULE_VALUE
}

object "ALERT_COMPANY\nCLIENT_CONFIGURATION" as ALERT_COMPANY {
    ISSUER_REF
}

object "EVENT_TEMPLATE" as EVENT_TEMPLATE {
    ISSUER_REF
    ADVICE_CODE
}

object ALERT_COUNTRY_CODE {
    ISSUER_REF
    SERVICE_TYPE
    COUNTRY_CODE
}

object ALERT_AREA_CODE {
    ISSUER_REF
    COUNTRY_CODE
}

object ALERT_OLWADVPRIO {
    ISSUER_REF
    ADVICE_CODE
    GROUP
}

ALERT_COMPANY "1" .. "*" EVENT_TEMPLATE
EVENT_TEMPLATE "1" .. "*" EVENT_CRITERIA
ALERT_COMPANY "1" --> "*" ALERT_COUNTRY_CODE
ALERT_COMPANY "1" --> "*" ALERT_AREA_CODE
ALERT_COUNTRY_CODE "1" . "*" ALERT_AREA_CODE
ALERT_COMPANY "1" . "*" ALERT_OLWADVPRIO

```

Product/Card-Program Configuration

```plantuml format="svg"

object ALERT_MASTER_DATA {
    ADVICE_CODE
    PRODUCT_REF
    ALERT_TEMPLATE_ID
    SERVICE_TYPE
}

object ALERT_TEMPLATES {
    ALERT_TEMPLATE_ID
    TEMPLATE_TYPE
}

object ALERT_CONFIGURATION {
    SERVICE_TYPE
    PRODUCT_REF
}

object ALERT_PACKET_CONFIG {
    ALERT_PACKET_ID
    PRODUCT_REF
}

object ALERT_PACKET_RULE {
    ALERT_PACKET_ID
    ADVICE_CODE
    AUTH_RESULT_GROUP
}

object AUTH_RESULT_LIST {
    AUTH_RESULT_GROUP
    AUTH_RESULT_PRIO
    AUTH_RESULT
    MATCH_ADVICE_CODE
    ALERT_COMPANY_ID
}


ALERT_MASTER_DATA "*" --> "1..2" ALERT_TEMPLATES
ALERT_PACKET_CONFIG "1" --> "*" ALERT_PACKET_RULE
ALERT_PACKET_RULE "*" -> "*" AUTH_RESULT_LIST
```

__Event Template__

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ISSUER_REF__        | VARCHAR2(50)   | not null   | Issuer reference |
| __ADVICE_CODE__       | VARCHAR2(50)   | not null   | Event identifier |
| EVENT_COMMENT         | VARCHAR2(200)  | null       | Event description |
| REF_SYSTEM            | VARCHAR2(10)   | not null   | System the event originates from<br>`CMS` = Card management system, `FO` = WLP-FO |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ISSUER_REF`, `ADVICE_CODE`    | primary | |
| `ISSUER_REF`, `REF_SYSTEM`     |         | Used to select all possible events for an event source system |

#### Notes

__Message Alert IBO__

  - Message Alert defines three global __service types__ (issuer independant):
      - alert (ALRT)
      - instalment (INST)
      - notification (NTFY)
  - __Events__ are (implicitely) defined per issuer
      - no event definition DB table
      - but __prioritization__ of events (`ALERT_OLWADVPRIO`) and __rules__ (`EVENT_CRITERIA`) are configured per issuer
  - __Events__ are assigned to a service type per card program (`ALERT_MASTER_DATA`)
  - __Events__ for the service type "alert" are grouped into __packets__ (`ALERT_PACKET_CONFIG`, `ALERT_PACKET_RULE`)
      - packets are defined per card program
  - __Templates__ are defined per issuer (or per card program?) (`ALERT_TEMPLATES`)
      - only by convention
      - "many to many" relationship between events and templates
      - Assignment of templates to an event per card program
  - Phone number validation configuration per issuer (`ALERT_COUNTRY_CODE`, `ALERT_AREA_CODE`)
      - but if a number gets validated is configured per card program (`ALERT_MASTER_DATA`)
  - __Authorization value__ validation per card program (packet) (`ALERT_PACKET_RULE`)
  - __Authorization result__ validation per issuer (`AUTH_RESULT_LIST`)
      - only by convention
  - Additional master data per card program for each service type (`ALERT_CONFIGURATION`)
  - Additional master data per issuer (`ALERT_COMPANY`)
  - `ALERT_FILE`, `ALERT_TEXT`

__Message Service IBO__

  - All configuration is done per issuer
      - __Events__ are defined per issuer (`EVENT_TEMPLATES`)
      - Event __rules__ are defined per issuer (`EVENT_CRITERIA`)
          - __but__ can be overriden by subscriptions (`SUBSCRIPTION_PARAMETER`)
      - Event field __mappings__ are defined per issuer (`EVENT_DATA_MAPPING`)
      - Master data per issuer (`CLIENT_CONFIGURATION`)
          - contains a list of fiids used by some stored procedures

__Integration__

  - Integrate Message Service events as new service type
      - or if possible as service type "notifications"
  - Merge `MESSAGE_SERVICE.EVENT_CRITERIA` into `MESSAGE.EVENT_CRITERIA`
  - Merge `MESSAGE_SERVICE.CLIENT_CONFIGURATION` into `MESSAGE.ALERT_COMPANY`
      - or into `MESSAGE.ALERT_CONFIGURATION`, if it should be card program + service type specific
  - Introduce new `EVENT_TEMPLATE` table into Message Alert
      - holds event configuration for issuer, e. g. event origin (CMS or WLP-FO), subscription level
      - should this be configured for issuer or card program?
          - if issuer specific, can parts be merged with `MESSAGE.ALERT_MASTER_DATA`?
      - maybe merge with `MESSAGE.ALERT_OLWADVPRIO`

### Subscriptions

!!! abstract
    __Message Alert__:
    
      - Card subscriptions for service types and packets (the later only applies for service type "alert")
      - Subscription services for Online Banking Portal (SOAP) and IBO (REST)
      - Subscriptions synchronized with IBO
      - Card information read from IBO during subscription
      - Calculation of fees are done by IBO

    __Message Service__:
    
      - Card/account/program subscriptions for events
      - Static subscriptions for card programs as part of the configuration
      - Subscription service for Online Banking Portal (SOAP) for card/account subscriptions
      - Card/account subscription can modify event criterias for this card/account

    __Integration__:

      - Card/account/program subscriptions for service types, packets and events
          - Packet subscription only applies for service type "alert"
          - Event subscription only applies for service type "Message Service"
          - New subscription type column in `ALERT_ACCOUNT`
          - New service type for "Message Service" events
      - Subscription for "Message Service" events specify the events t
      - Card/account subscription can modify event criterias for "Message Service" events for this card/account
          - New table `SUBSCRIPTION_PARAMETER` under `ALERT_ACCOUNT`
      - Subscription services for Online Banking Portal (SOAP) and IBO (REST) for card/account subscriptions
      - Subscriptions synchronized with IBO based on issuer configuration (`ALERT_COMPANY`) and service type

!!! question
    __Q1__

    :   Is it possible to change the Message Service event subscriptions to packet subscriptions?

    __Q2__

    :   Is it necessary to support subscription criterias (only used by Postbank)?

#### Database Tables

```plantuml format="svg"

object "ALERT_ACCOUNT\nSUBSCRIPTION" as SUBSCRIPTION {
    **ALERT_ACCOUNT_ID (PK)**
    ISSUER_REF
    TYPE
    REFERENCE
}

object SUBSCR_SERVICE {
    **SUBSCR_SERVICE_ID (PK)**
    **ALERT_ACCOUNT_ID (FK)**
    SERVICE
    PACKET_ID (FK)
}

object "ALERT_ADDRESS\nSUBSCRIPTION_ADDRESS" as SUBSCRIPTION_ADDRESS {
    **ALERT_ADDRESS_ID (PK)**
    **SUBSCR_SERVICE_ID (FK)**
    TYPE
    ADDRESS
}

object SUBSCR_EVENT {
    **SUBSCR_EVENT_ID (PK)**
    **SUBSCR_SERVICE_ID (FK)**
    ADVICE_CODE
}

object SUBSCR_PARAMETER {
    **SUBSCR_PARAMETER_ID (PK)**
    **SUBSCR_EVENT_ID (FK)**
    ADVICE_CODE
    CRITERIA_ENTRY
    RULE_OPERATOR
    RULE_VALUE
}

object ALERT_PACKET_CONFIG #WhiteSmoke {
    **ALERT_PACKET_ID (PK)**
    **PRODUCT_REF**
    PACKET_NAME
    PACKET_FEE
}

SUBSCRIPTION "1" --> "*" SUBSCR_SERVICE
SUBSCR_SERVICE "1" --> "*" SUBSCRIPTION_ADDRESS
SUBSCR_SERVICE "1" --> "*" SUBSCR_EVENT
SUBSCR_EVENT "1" --> "*" SUBSCR_PARAMETER
SUBSCR_SERVICE "0" -> "1" ALERT_PACKET_CONFIG

```

__ALERT_ACCOUNT / SUBSCRIPTION__

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_ACCOUNT_ID__  | NUMBER(9)      | not null   | Sequence |
| REFERENCE             | VARCHAR2(50)   | not null   | Card contract reference (for type `CARD`), account reference (for type `ACCT`), card product (program) reference (for type `PROD`), card number range (for type `RANG`) |
| TYPE                  | CHAR(4)        | not null   | `ACCT` = account related, `CARD` = card related, `PROD` = product (program) related, `RANG` = Card range related |
| PRODUCT_REF           | VARCHAR2(50)   | not null   | Card product (program) reference |
| ISSUER_REF            | VARCHAR2(50)   | not null   | Issuer reference |
| SOURCE                | CHAR(3)        | not null   | `ONL` = Online Banking, `IBO` = IBO, `CFG` = Configuration |
| CREATE_DATE           | TIMESTAMP      | not null   |  |

| __Index__                         | Type    | Description |
| --------------------------------- | ------- | ----------- |
| `ALERT_ACCOUNT_ID`                | primary | |
| `ISSUER_REF`, `TYPE`, `REFERENCE` | unique  | Find subscription for issuer by type and reference |

__SUBSCR_SERVICE__

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_ACCOUNT_ID__  | NUMBER(9)      | not null   | Foreign key for `ALERT_ACCOUNT` |
| __SERVICE_TYPE__      | CHAR(4)        | not null   | `ALRT` = Alert, `INST` = Instalment, `NTFY` = Notification, `MSGS` = Message Service |
| ALERT_PACKET_ID       | NUMBER(9)      | null       | Foreign key for `ALERT_PACKET_CONFIG`, only for `SERVICE_TYPE` = `ALRT`) |
| SOURCE                | CHAR(3)        | not null   | `ONL` = Online Banking, `IBO` = IBO, `CFG` = Configuration |
| CREATE_DATE           | TIMESTAMP      | not null   |  |
| UPDATE_DATE           | TIMESTAMP      | not null   |  |

| __Index__                          | Type    | Description |
| ---------------------------------- | ------- | ----------- |
| `ALERT_ACCOUNT_ID`, `SERVICE_TYPE` | primary | Only one subscription for each service type allowed |
| `ALERT_ACCOUNT_ID`                 | foreign | Reference to `ALERT_ACCOUNT` |
| `ALERT_PACKET_ID`                  | foreign | Reference to `ALERT_PACKET_CONFIG` |


__ALERT_ADDRESS / SUBSCR_ADDRESS__

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_ACCOUNT_ID__  | NUMBER(9)      | not null   | Foreign key for `ALERT_ACCOUNT` |
| __ADDRESS_SOURCE__      | CHAR(4)        | not null   | Foreign key for `SUBSCR_SERVICE` (field `SERVICE_TYPE`) |
| __ADDRESS_TYPE__      | CHAR(1)        | not null   | `M` = MSISDN, `E` = email |
| ADDRESS               | VARCHAR2(256)  | not null   | Destination address for this address type |
| SOURCE                | CHAR(3)        | not null   | `ONL` = Online Banking, `IBO` = IBO, `CFG` = Configuration |
| CREATE_DATE           | TIMESTAMP      | not null   |  |
| UPDATE_DATE           | TIMESTAMP      | not null   |  |

| __Index__                           | Type    | Description |
| ----------------------------------- | ------- | ----------- |
| `ALERT_ACCOUNT_ID`, `ADDRESS_SOURCE`, `ADDRESS_TYPE` | primary  | Only one address for each address type allowed |
| `ALERT_ACCOUNT_ID`                  | foreign | Foreign key for `ALERT_ACCOUNT` |
| `ALERT_ACCOUNT_ID`, `ADDRESS_SOURCE`  | foreign | Foreign key for `SUBSCR_SERVICE` |


__SUBSCR_EVENT__

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __SUBSCR_EVENT_ID__   | NUMBER(9)      | not null   | Sequence |
| SUBSCR_SERVICE_ID     | NUMBER(9)      | not null   | Foreign key for `SUBSCR_SERVICE` |
| ADVICE_CODE           | VARCHAR2(50)   | not null   | Event identifier |
| CREATE_DATE           | TIMESTAMP      | not null   |  |
| UPDATE_DATE           | TIMESTAMP      | not null   |  |

| __Index__                          | Type    | Description |
| ---------------------------------- | ------- | ----------- |
| `SUBSCR_EVENT_ID`                  | primary | |
| `SUBSCR_SERVICE_ID`                | foreign | |
| `SUBSCR_SERVICE_ID`, `ADVICE_CODE` | unique  | Only one subscription for each event allowed |


__SUBSCR_PARAMETER__

| __Column__              | Type(Size)     | Constraint | Description |
| ----------------------- | -------------- | ---------- | ----------- |
| __SUBSCR_PARAMETER_ID__ | NUMBER(9)      | not null   | Sequence |
| ...                     | ...            | ...        | ... |

#### Notes

__Message Alert IBO__

  - __Subscriptions__ for __cards__ (card contracts)
  - Card subscriptions for __service types__ (alert, instalment, notification)
    - For service type "alert" subscription is done additionally for a __packet__
  - Subscriptions are managed by the __Online Banking Portal__ (SOAP-Service) or __PremiumGUI__ (REST-Service)
  - Cards are validated against IBO and missing card information is retrieved from IBO (CARD_CONTRACT_REF, PRODUCT_REF, ISSUER_REF)
  - Subscriptions are synced with IBO
      - Subscription fees will be calculated by IBO

__Message Service IBO__

Part 1 (NN, OP - Card Control):

  - __Subscriptions__ for __card programs__ (card number ranges)
  - Card program subscriptions for __events__
  - __Subscription is static__ and part of the configuration

Part 2 (Postbank, ING):

  - __Subscriptions__ for __cards__ and __accounts__
      - maybe for companies?
  - Account and card subscriptions for events can modify the __event criterias as part of the subscription__
  - Account and card subscriptions are managed by the __Online Banking Portal__ (SOAP-Service)
      - Card program subscriptions are still static

__Integration__

  - Message Service events are assigned to a new service type, subscriptions can be done for this new service type
      - For Message Service subscriptions the events have to be specified that belong to this subscription
      - Introduce new `SUBSCRIPTION_SERVICE` table (clean up)
      - Introduce new `SUBSCRIPTION_SERVICE_EVENT` table
      - Introduce new `SUBSCRIPTION_SERVICE_PARAMETER` table
  - Issuer configuration defines if subscriptions are synced with IBO (`ALERT_COMPANY`)
  - Subscriptions will be extended to support different types (card, account, card program, company)
      - Merge of `MESSAGE_SERVICE.SUBSCRIPTIONS` into `MESSAGE.ALERT_ACCOUNT`
      - Maybe a new columns to separate subscriptions from configuration from others?
  - Card program subscriptions could be split into subscriptions for card number ranges and product reference
  - Output channel for Message Service events can be modelled as a new address type (`ALERT_ADDRESS`) (address types would be email, msisdn, message-service and `ALERT_COMPANY` would define what output channel will be used for "message-service" - e. g. ING_QUEUE, POSTBANK_WS)
      - Another approach could be to only introduce the new address type in `ALERT_MASTER_DATA.ADDRESS_SOURCE`



### Event Processing

#### Message Alert IBO

  - WLP-FO (head-release): STNI-switchouts read from MQ-Series Queues
  - IBO / GODS: CMS-events (Batch-SMS), Instalment
  - External systems (CSP, Issuer TSP?)
  - SMS-reception

#### Message Service IBO

Part 1:

  - WLP-FO (head-release - separate instance): STNI-switchouts read from MQ-Series Queues
  - No dependency to a CMS
      - No card contract reference and no product references

Phase 2:

  - IBO / GODS: CMS-events

#### Integration Strategies

WLP-FO (STNI):

  - Check against `EVENT_CRITERIA` to identify events that do not originate from Online Watcher
      - Check is done for the criterias defined for the issuer
      - Criterias overwritten by the subscription have to be checked later in the processing chain
      - This assumes, that a subscription criteria can only result to fewer events and not to more events!
  - Check of the Online Watcher events against the priority table `ALERT_OLWADVPRIO`

CMS:

  - Batch triggered generation of events from IBO / GODS queries
  - Check against `EVENT_CRITERIA` to identify events

Event Processing:

  - Check if subscription for card exists (`ALERT_ACCOUNT`)
      - Note: Message Alert reads the `PRODUCT_REF` from this subscription entry!
  - Get service type for event from `ALERT_MASTER_DATA` (fetch by `PRODUCT_REF` and `ADVICE_CODE`)
  - For service type "alert" additional check for packet rules (auth-result and auth-value)
      - This can change the event / advice code (requires reloading of configuration)
  - Check if subscription for this service type exists (`ALERT_ADDRESS`)
  - Get configuration for service type (`ALERT_CONFIGURATION`)
  - If address type is msisdn and phone number should be validated (`ALERT_MASTER_DATA.CHECK_COUNTRY_CODE`), validate phone number (`ALERT_COUNTRY_CODE`, `ALERT_AREA_CODE`) 
  - Fetch message templates (`ALERT_TEMPLATES`)
  - Create message (text, subject, ...)
  - Deliver message to destination address and write `MESSAGE_LOG`

!!! question
    __Q1__

    :   At which point should the `EVENT_LOG` be written?

### Output Channels

#### Message Alert IBO

  - Support for SMS and email
      - Templates with variables and functions used to create message body
      - Support for different SMS provider

#### Message Service IBO

  - Support for web service calls and queues (`CLIENT_CONFIGURATION.SEND_PROC`)
      - Message body is JSON constructed from configuration (`EVENT_DATA_MAPPING`)

#### Integration Strategies

  - Reuse template mechanism to construct JSON payload?
  - Introduce `EVENT_DATA_MAPPING` for output channel `queue` / `ws`?