# Data Model

- Specification: `S:\Daten\PROJEKTE\SMS\003 Message Alert - Copernicus`
  (files: `Message Alert Datenbanktabellen *`)

## Issuer Configuration

```plantuml format="svg"

object "ALERT_COMPANY" as ALERT_COMPANY {
    **ALERT_COMPANY_ID**
    ISSUER_REF
    COMPANY_REF
    COMPANY_NAME
    COUNTRY_CODE
    SAP_REF
    SEND_PROC
    CREATE_DATE
}

object "EVENT_TEMPLATE" as EVENT_TEMPLATE {
    **ALERT_COMPANY_ID**
    **EVENT_NAME**
    ACTION_CODE
    ADVICE_CODE
    SERVICE_TYPE
    TYPE
    REF_SYSTEM
}

object EVENT_CRITERIA {
    **ALERT_COMPANY_ID**
    **EVENT_NAME**
    **CRITERIA_ENTRY**
    OPERATOR
    VALUE
    VALUE_FORMAT
}

object EVENT_DATA_MAPPING {
    **ALERT_COMPANY_ID**
    **EVENT_NAME**
    **EVENT_DATA_KEY**
    DATA_ACCESS_KEY
    DEFAULT_VALUE
    DATA_TYPE
    DATA_FORMAT
    INPUT_FORMAT
}

object ALERT_OLWADVPRIO {
    **ALERT_OLWADVPRIO_ID**
    ALERT_COMPANY_ID
    OLW_ADVICE_CODE
    PRIORITY
    PRIORITY_GROUP
    CREATE_DATE
}

ALERT_COMPANY "1" --> "*" EVENT_TEMPLATE
EVENT_TEMPLATE "1" --> "*" EVENT_CRITERIA
EVENT_TEMPLATE "1" --> "*" EVENT_DATA_MAPPING
ALERT_COMPANY "1" --> "*" ALERT_OLWADVPRIO

```

### ALERT_COMPANY

`ALERT_COMPANY` holds the configuration data for a company / issuer.
A database package provides the configuration, the application only
reads the data.

| Column                | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_COMPANY_ID__  | NUMBER         | not null   | Message Alert specific company id |
| ISSUER_REF            | VARCHAR2(50)   | not        | Issuer reference from IBO and WLP-FO (`null` if company does not use IBO as CMS) |
| COMPANY_REF           | VARCHAR2(50)   | not null   | Unique company identifier (for companies using IBO as CMS identical to `ISSUER_REF`) |
| COMPANY_NAME          | VARCHAR2(50)   | not null   |  |
| COUNTRY_CODE          | VARCHAR2(2)    | not null   | ISO-3166 Alpha-2 Country Code |
| SAP_REF               | VARCHAR2(50)  | null        | Reference number for billing |
| SEND_PROC             | VARCHAR2(20)  | null        | Output channel for message service events<br>`POSTBANK_WS` for Postbank, `ING_QUEUE` for ING |
| CREATE_DATE           | DATE          | not null    |  |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_COMPANY_ID`             | primary | Sequence |
| `ISSUER_REF`                   | unique  |  |
| `COMPANY_REF`                  | unique  |  |

### EVENT_TEMPLATE

`EVENT_TEMPLATE` holds the configuration data that defines which events (apart
from the Online Watcher events configured in `ALERT_OLWADVPRIO`) exist for a
company / issuer.

An event is assigned to a service type and a subscription type.
A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_COMPANY_ID__  | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| __EVENT_NAME__        | VARCHAR2(50)   | not null   | Event identifier |
| ACTION_CODE           | VARCHAR2(50)   | null       | External event identifier |
| ADVICE_CODE           | VARCHAR2(50)   | null       | Advice code |
| SERVICE_TYPE          | VARCHAR2(32)   | not null   | Service type the event belongs to (`ALRT`, `INST`, `NTFY`, `MSGS`) |
| TYPE                  | CHAR(4)        | not null   | Subscription type the event belongs to (`ACCT`, `CARD`, `PROD`, `COMP`, `RANG`) |
| REF_SYSTEM            | VARCHAR2(10)   | not null   | System the event originates from<br>`CMS` = Card management system, `FO` = WLP-FO, `GODS` = Issuer Data Repository (GODS) |
| EVENT_COMMENT         | VARCHAR2(200)  | null       | Event description |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_COMPANY_ID`, `EVENT_NAME`     | primary | |
| `ALERT_COMPANY_ID`             | foreign | Reference to `ALERT_COMPANY` |
| `ALERT_COMPANY_ID`, `REF_SYSTEM`     |         | Used to select all possible events for an event source system |

!!! question "Open questions"

      - Column `CREATE_DATE` is missing?

### EVENT_CRITERIA

`EVENT_CRITERIA` holds the configuration to identify if an incoming event matches
an event from `EVENT_TEMPLATE`.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_COMPANY_ID__  | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| __EVENT_NAME__        | VARCHAR2(50)   | not null   | Event identifier |
| __CRITERIA_ENTRY__    | VARCHAR2(50)   | not null   | Criteria identifier |
| OPERATOR              | VARCHAR2(30)   | not null   | Comparison operator |
| VALUE                 | VARCHAR2(30)   | null       | Comparison value |
| VALUE_FORMAT          | VARCHAR2(10)   | null       | Value type: `n` (numeric), `c` (string) or `d` (timestamp) |
| CRITERIA_COMMENT      | VARCHAR2(200)  | null       |  |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_COMPANY_ID`, `EVENT_NAME`, `CRITERIA_ENTRY`     | primary | |
| `ALERT_COMPANY_ID`,            | foreign | Referencens `ALERT_COMPANY` |
| `ALERT_COMPANY_ID`, `EVENT_NAME`  | foreign | Referencens `EVENT_TEMPLATE` |

### ALERT_OLWADVPRIO

`ALERT_OLWADVPRIO` holds the configuration data to identify relevant Online
Watcher (OLW) advice codes in an STNI-request.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_OLWADVPRIO_ID__  | NUMBER(6)   | not null   | Sequence |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| OLW_ADVICE_CODE       | VARCHAR2(35)   | not null   | Online Watcher advice code |
| PRIORITY              | NUMBER(2)      | not null   | Priority of the advice code |
| PRIORTIY_GROUP        | VARCHAR2(2)    | not null   | Group of the advice code |
| CREATE_DATE           | DATE           | null       |  |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_OLWADVPRIO_ID`          | primary | Sequence    |
| `ALERT_COMPANY_ID`             | foreign | Foreign key to `ALERT_COMPANY`    |
| `ALERT_COMPANY_ID`, `OLW_ADVICE_CODE` | unique | Only one advice code for each company allowed    |

## Message Service Configuration

### EVENT_DATA_MAPPING

`EVENT_DATA_MAPPING` holds the configuration for the outgoing event. It contains
the information which field in the incoming event will be mapped into the
outgoing event.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_COMPANY_ID__  | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| __EVENT_NAME__        | VARCHAR2(50)   | not null   | Event identifier from `EVENT_TEMPLATE` |
| __EVENT_DATA_KEY__    | VARCHAR2(36)   | not null   | Field name in the outgoing event |
| DATA_ACCESS_KEY       | VARCHAR2(200)  | null       | Field identifiere in the incoming event |
| DEFAULT_VALUE         | VARCHAR2(200)  | null       | Default value if the `DATA_ACCESS_KEY` cannot be found in the incoming event |
| DATA_TYPE             | VARCHAR2(6)    | null       | `n` = numeric, `c`= alphanumeric, `d` = date |
| DATA_FORMAT           | VARCHAR2(36)   | null       | Value format for the outgoing event |
| INPUT_FORMAT          | VARCHAR2(36)   | null       | Value format of the incoming event |
| MAPPING_COMMENT       | VARCHAR2(80)   | null       |  |
| EVENT_DATA_YN         | CHAR(1)        | null       | `Y` = include in outgoing event, `N` = do not include in outgoing event |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_COMPANY_ID`, `EVENT_NAME`, `EVENT_DATA_KEY`  | primary |    |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY`    |
| `ALERT_COMPANY_ID`, `EVENT_NAME` | foreign | Foreign key for `EVENT_TEMPLATE`    |

## Alert Configuration

```plantuml format="svg"

object "ALERT_COMPANY" as ALERT_COMPANY #LightGrey {
    **ALERT_COMPANY_ID**
}

object ALERT_MASTERDATA {
    **ALERT_MASTERDATA_ID**
    ALERT_COMPANY_ID
    ADVICE_CODE
    PROCESSING
    ALERT_SERVICE_TYPE
    ADDRESS_SOURCE
    CHECK_AUTHRESULT
    CHECK_COUNTRY_CODE
}

object ALERT_EVENTMAP {
    **ALERT_EVENTMAP_ID**
    ALERT_COMPANY_ID
    PRODUCT_REF
    ADVICE_CODE
    ALERT_TEMPLATE_ID
}

object ALERT_COUNTRY_CODE {
    ALERT_COMPANY_ID
    ALERT_SERVICE_TYPE
    COUNTRY_CODE
}

object ALERT_AREA_CODE {
    ALERT_COMPANY_ID
    COUNTRY_CODE
    AREA_CODE
}

object ALERT_TEMPLATES {
    ALERT_TEMPLATE_ID
    TEMPLATE_TYPE
    ALERT_COMPANY_ID
}

object ALERT_CONFIGURATION {
    **ALERT_CONFIGURATION_ID**
    ALERT_COMPANY_ID
    PRODUCT_REF
    ALERT_SERVICE_TYPE
}

object ALERT_PARAMETER {
    **ALERT_PARAMETER_ID**
    ALERT_COMPANY_ID
    PRODUCT_REF
}

object ALERT_PACKET_CONFIG {
    **ALERT_PACKET_ID**
    ALERT_COMPANY_ID
    PRODUCT_REF
    PACKET_NAME
}

object ALERT_PACKET_RULE {
    **ALERT_PACKET_ID**
    **ADVICE_CODE**
    ALERT_COMPANY_ID
    AUTH_RESULT_GROUP
}

object AUTH_RESULT_LIST {
    AUTH_RESULT_GROUP
    ALERT_COMPANY_ID
    AUTH_RESULT_PRIO
    AUTH_RESULT
    MATCH_ADVICE_CODE
    ALERT_COMPANY_ID
}

ALERT_COMPANY "1" --> "*" ALERT_MASTERDATA
ALERT_MASTERDATA "1" --> "*" ALERT_EVENTMAP
ALERT_EVENTMAP "*" --> "1..2" ALERT_TEMPLATES
ALERT_PACKET_CONFIG "1" --> "*" ALERT_PACKET_RULE
ALERT_PACKET_RULE "*" -> "*" AUTH_RESULT_LIST
ALERT_COMPANY "1" --> "*" ALERT_COUNTRY_CODE
ALERT_COMPANY "1" --> "*" ALERT_AREA_CODE
ALERT_COUNTRY_CODE "*" . "*" ALERT_AREA_CODE
ALERT_COMPANY "1" --> "*" ALERT_CONFIGURATION
ALERT_COMPANY "1" --> "*" ALERT_PARAMETER
ALERT_COMPANY "1" --> "*" ALERT_PACKET_CONFIG
```

### ALERT_MASTERDATA

`ALERT_MASTERDATA` holds the configuration for the alert events for a company / issuer.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_MASTERDATA_ID__ | NUMBER(6)    | not null   | Unique identifier for this table, filled from sequence |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| ADVICE_CODE           | VARCHAR2(35)   | not null   | Unique together with `ALERT_COMPANY_ID`, identifier for an alert event |
| PROCESSING            | CHAR(4)        | null       | `ALRT` = Alert (Packet Filter) `INST` = Installment (Offer) `STD` = no special processing |
| ALERT_SERVICE_TYPE    | CHAR(1)        | null       | Used for selection of entry in table `ALERT_CONFIGURATION`<br>`A` = Alert, `I` = Instalment, `N` = Notification |
| REQUEST_TYPE          | CHAR(1)        | null       | (Currently unused)<br>`A` = Alert, ... |
| ADDRESS_SOURCE        | CHAR(4)        | null       | `EVNT` = Event->contact type=msisdn, Event->contact type=email, `ALRT` = Address usage Message Alert, `INST` = Address usage Installment, `NTFY` = Address usage Notification |
| CHECK_AUTHRESULT      | CHAR(1)        | null       | `Y` = Check Auth Result, `N` = no Check |
| CHECK_COUNTRY_CODE    | CHAR(1)        | null       | `Y` = Check Alert Country Code, `N` = no Check |
| DESCRIPTION           | VARCHAR2(60)   | null       | Short description of the event |
| LONG_DESC             | VARCHAR2(240)  | null       | Long description of the event |
| CREATE_DATE           | DATE           | null       | Creation date of the record from SYSDATE |
| UPDATE_DATE           | DATE           | null       | Date of last update of the record |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_MASTERDATA_ID`          | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |
| `ALERT_COMPANY_ID`, `ADVICE_CODE`   | unique  | |

### ALERT_EVENTMAP

`ALERT_EVENTMAP` holds the configuration for the outgoing alert events. The configuration
is done per card program (product).

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_EVENTMAP_ID__ | NUMBER(6)      | not null   | Unique identifier for this table, filled from sequence |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| PRODUCT_REF           | VARCHAR2(50)   | not null   | Product reference (card program) |
| ADVICE_CODE           | VARCHAR2(35)   | not null   | Identifier for an alert event |
| ALERT_TEMPLATE_ID     | NUMBER(6)      | not null   | Reference to `ALERT_TEMPLATE` |
| DESCRIPTION           | VARCHAR2(60)   | null       | Short description of the event |
| LONG_DESC             | VARCHAR2(240)  | null       | Long description of the event |
| CREATE_DATE           | DATE           | null       | Creation date of the record from SYSDATE |
| UPDATE_DATE           | DATE           | null       | Date of last update of the record |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_EVENTMAP_ID`            | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |
| `ALERT_TEMPLATE_ID`            |  | Reference to `ALERT_TEMPLATE` entries |
| `ALERT_COMPANY_ID`, `ADVICE_CODE`   | unique  | |

### ALERT_COUNTRY_CODE

`ALERT_COUNTRY_CODE` holds the configuration for the valid country codes for msisdns
for outgoing SMS for alert events. Configuration is done for a company / issuer
and the different alert service types.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_COMPANY_ID__  | NUMBER(6)      | not null   | Company reference |
| __ALERT_SERVICE_TYPE__ | VARCHAR2(1)    | not null   | Alert service type (`A`, `I`, `M`) |
| __COUNTRY_CODE__      | VARCHAR2(5)    | not null   | Country code, e. g. `+49` |
| CHECK_AREA_CODE_YN    | CHAR(1)        | not null   | `Y` (yes) or `N` (no) |
| DEFAULT_YN            | CHAR(1)        | not null   | `Y` (yes) or `N` (no) |
| CREATE_DATE           | DATE           | null       |  |
| UPDATE_DATE           | DATE           | null       |  |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_COMPANY_ID`, `ALERT_SERVICE_TYPE`, `COUNTRY_CODE`     | unique | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |

### ALERT_AREA_CODE

`ALERT_AREA_CODE` holds the configuration for the valid area codes for msisdns
for outgoing SMS for alert events. Configuration is done for a company / issuer
and country code.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_COMPANY_ID__  | NUMBER(6)      | not null   | Company reference |
| __COUNTRY_CODE__      | VARCHAR2(5)    | not null   | Country code, e. g. `+49` |
| __AREA_CODE__         | VARCHAR2(3)    | not null   | Area code |
| MSISDN_MIN            | NUMBER(3)      | null       | Minimum number of digits including country and area code |
| MSISDN_MAX            | NUMBER(3)      | null       | Maximum number of digits including country and area code |
| CREATE_DATE           | DATE           | null       |  |
| UPDATE_DATE           | DATE           | null       |  |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_COMPANY_ID`, `COUNTRY_CODE`, `AREA_CODE` (add?)    | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |

### ALERT_CONFIGURATION

`ALERT_CONFIGURATION` holds the configuration for processing of alert events for
card programs (products).

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_CONFIGURATION_ID__ | NUMBER(6) | not null   | Unique ID, derived from sequence |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| PRODUCT_REF           | VARCHAR2(50)   | not null   | Card program identification from IBO |
| CREATE_DATE           | DATE           | not null   | Creation date of record |
| ALERT_SERVICE_TYPE    | CHAR(1)        | not null   | A = Alert,I = Instalment,N = Notification |
| EMAIL_ADDRESS         | VARCHAR2(60)   | null       | Email address in field mail-from (SMTP protocol) |
| EMAIL_FROM            | VARCHAR2(60)   | null       | Email address in field from (if empty: value in column EMAIL_SENDER is used) |
| EMAIL_SENDER          | VARCHAR2(60)   | null       | Email address in field sender (if empty: no value set in message) |
| EMAIL_REPLY_TO        | VARCHAR2(60)   | null       | Email address in field reply-to (if empty: no value set in message) |
| WELCOME_SMS_ADVICE_CODE | VARCHAR2(35) | null       | Defines access to Welcome text to be sent after registration (cp. table alert_masterdata). |
| CHANGE_SMS_ADVICE_CODE  | VARCHAR2(35) | null       | Defines access to change advice text to be sent after a change in registration (cp. table alert_masterdata). |
| GOODBYE_SMS_ADVICE_CODE | VARCHAR2(35) | null       | Defines access to goodbye text to be sent after deregistration (cp. table alert_masterdata). |
| WELCOME_EMAIL_ADVICE_CODE | VARCHAR2(35) | null     | Defines access to welcome email text to be sent after registration (cp. table alert_masterdata). |
| CHANGE_EMAIL_ADVICE_CODE | VARCHAR2(35) | null      | Defines access to change email text to be sent after a change in registration (cp. table alert_masterdata) |
| GOODBYE_EMAIL_ADVICE_CODE | VARCHAR2(35) | null     | Defines access to goodbye email text to be sent after deregistration (cp. table alert_masterdata). |
| SENDER_ID             | VARCHAR2(30)   | null       | Identifier for the sender of SMS |
| SERVICE_PROVIDER      | VARCHAR2(35)   | not null   | Identifier for SMS service provider: `MM` = Message Mobile, `MV` = MobileView, `SAP` = SAP |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_CONFIGURATION_ID`       | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |
| `PRODUCT_REF`, `ALERT_SERVICE_TYPE`  | unique  | |

### ALERT_PARAMETER

`ALERT_PARAMETER` holds the configuration for additional template variables for
card programs (products).

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_PARAMETER_ID__ | NUMBER(6) | not null   | Unique ID, derived from sequence |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| PRODUCT_REF           | VARCHAR2(50)   | not null   | Card program identification from IBO |
| CREATE_DATE           | DATE           | not null   | Creation date of record |
| PROGRAM_NAME          | VARCHAR2(50)   | null       | name of the card program, `${WLSI_PROGRAM_NAME}` |
| BANK_NAME             | VARCHAR2(40)   | null       | name of the bank, `${WLSI_BANK_NAME}` |
| LONG_PROGRAM_NAME     | VARCHAR2(80)   | null       | name of the card program - longer version, `${WLSI_LONG_PROGRAM_NAME}` |
| LONG_BANK_NAME        | VARCHAR2(80)   | null       | name of the bank - longer version, `${WLSI_LONG_BANK_NAME}` |
| KIS_PHONE_NUMBER      | VARCHAR2(23)   | null       | KIS telephone number, `${WLSI_KIS_PHONE_NUMBER}` |
| FAX_NUMBER            | VARCHAR2(23)   | null       | Field fax number reference to `${WLSI_FAX_NUMBER}` |
| BANK_PHONE_NUMBER     | VARCHAR2(23)   | null       | Field bank phone number reference to `${WLSI_BANK_PHONE_NUMBER}` |
| BANK_FAX_NUMBER       | VARCHAR2(23)   | null       | Field bank fax number reference to `${WLSI_BANK_FAX_NUMBER}` |
| BANK_EMAIL_ADDRESS    | VARCHAR2(60)   | null       | Field bank email address reference to `${WLSI_BANK_EMAIL_ADDRESS}` |
| WEB_ADDRESS           | VARCHAR2(256)  | null       | Web address of the bank, `${WLSI_WEB_ADDRESS}` |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_PARAMETER_ID`           | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |
| `PRODUCT_REF`, `SERVICE_TYPE`  | unique  | |

### ALERT_TEMPLATES

`ALERT_TEMPLATES` holds the configuration for the outgoing alert events.

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_TEMPLATE_ID__ | NUMBER(6) | not null | Identifier for the template, filled with a sequence |
| TEMPLATE_TYPE | CHAR(1) | not null | `M` = Mobile-SMS, `E` = Email unique with TEMPLATE_NAME and VALID_FROM |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| TEMPLATE_NAME | VARCHAR2(40) | not null | name of the template, unique with TEMPLATE_TYPE and VALID_FROM |
| TEMPLATE | VARCHAR2(4000) | null |     The SMS template or text email Template incl. the placeholder in the Shape ${} |
| HTML_TEMPLATE | CLOB | null |     HTML email template including the Placeholder in the form of ${} |
| SUBJECT | VARCHAR2(255) | null |     Email subject, empty in the case of SMS |
| VALIDITY_PERIOD | NUMBER(5) | null |     The time in minutes the SMS should be tried to deliver. |
| START_TIME | VARCHAR2(8) | null |     time at which at least one asynchronous SMS is sent. Format = HH:MM:SS |
| END_TIME | VARCHAR2(8) | null |     Time until which an asynchronous SMS is sent at the latest Format = HH:MM:SS |
| VALID_FROM | DATE | not null | Earliest date on which this template is valid.unique with TEMPLATE_TYPE and TEMPLATE_NAME |
| CREATOR_ID | VARCHAR2(80) | not null | Identification of the change of a template |
| CREATE_DATE | DATE | null |creation date/time of the record |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `TEMPLATE_NAME`, `TEMPLATE_TYPE`, `VALID_FROM`   | primary  | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |

### ALERT_TEXT

TODO

### ALERT_FILE

TODO

### ALERT_PACKET_CONFIG

`ALERT_PACKET_CONFIG` holds the configuration for the alert packets for a card
program (product).

A database package provides the configuration, the application only
reads the data.

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_PACKET_ID__ | NUMBER(9) | not null | Generated unique identifier of service packet (cp. sequence seq_alert_packet_config) |
| PACKET_NAME | VARCHAR2(3) | not null | Name of service packet |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| PRODUCT_REF | VARCHAR2(50) | not null | Identification card program from IBO |
| PACKET_ORDER | NUMBER(2) | null | Display order of service packets to be used in GUI application |
| PACKET_FEE_AMT | NUMBER(13,2) | null | Current fee amount of service packet (EUR) |
| CREATE_DATE | DATE | null | Creation date of physical record |
| UPDATE_DATE | DATE | null | Last update date of physical record |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_PACKET_ID`              | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |
| `ALERT_COMPANY_ID`, `PACKET_NAME`, `PRODUCT_REF`   |  | |

### ALERT_PACKET_RULE

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __ALERT_PACKET_ID__ | NUMBER(6) | not null | identifier of service packet (cp. table alert_packet_config) together with advice_code primary key |
| ALERT_COMPANY_ID      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| __ADVICE_CODE__ | VARCHAR2(35) | not null | Advice Code, together with alert_packet_id primary key |
| AUTH_VALUE_DEF | NUMBER(8) | null | Current threshold for the authorization amount (EUR) |
| CREATE_DATE | DATE | null | Creation date of physical record |
| UPDATE_DATE | DATE | null | Last update date of physical record |
| AUTH_RESULT_GROUP | VARCHAR2(35) | null | reference to field auth_result_group in  table auth_result_group |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `ALERT_PACKET_ID`, `ADVICE_CODE` | primary | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |
| `ALERT_COMPANY_ID`, `ALERT_PACKET_ID`, `ADVICE_CODE`, `AUTH_VALUE_DEF`   |  | |

### AUTH_RESULT_LIST

| __Column__            | Type(Size)     | Constraint | Description |
| --------------------- | -------------- | ---------- | ----------- |
| __AUTH_RESULT_GROUP__ | VARCHAR2(35) | not null | Group name of the list of valid values in authResult. |
| __ALERT_COMPANY_ID__      | NUMBER(6)      | not null   | Reference to `ALERT_COMPANY` |
| AUTH_RESULT_PRIO | NUMBER(2) | not null | Priority entry, highest level: 1 |
| __AUTH_RESULT__ | VARCHAR2(40) | not null | Vvalue with would lead to the dispatch of a message. |
| CREATE_DATE | DATE | null | Creation date of physical record. |
| UPDATE_DATE | DATE | null | When the row is updated, sysdate is stored here. |
| MATCH_ADVICE_CODE | VARCHAR2(35) | null | if set: is used together with the current value for PRODUCT_REF as index in the Table ALERT_MASTERDATA for the message to be sent is used. |
| ALERT_COMPANY_ID | NUMBER(6) | not null | reference to table alert_company |

| __Index__                      | Type    | Description |
| ------------------------------ | ------- | ----------- |
| `AUTH_RESULT_GROUP`, `AUTH_RESULT`   | primary  | |
| `ALERT_COMPANY_ID`             | foreign | Foreign key for `ALERT_COMPANY` |

## Subscriptions

```plantuml format="svg"

object "SUBSCRIPTION" as SUBSCRIPTION {
    **SUBSCRIPTION_ID**
    REFERENCE
    TYPE
    ALERT_COMPANY_ID
    PRODUCT_REF
}

object SUBSCRIPTION_SERVICE {
    **SUBSCRIPTION_SERVICE_ID**
    ALERT_COMPANY_ID
    SUBSCRIPTION_ID
    SERVICE_TYPE
    PACKET_ID
}

object "ALERT_ADDRESS" as SUBSCRIPTION_ADDRESS {
    **ALERT_ADDRESS_ID**
    ALERT_COMPANY_ID
    ADDRESS_TYPE
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

SUBSCRIPTION "1" --> "*" SUBSCRIPTION_SERVICE
SUBSCRIPTION "1" --> "*" SUBSCRIPTION_ADDRESS
SUBSCRIPTION_SERVICE "1" --> "*" SUBSCR_EVENT
SUBSCR_EVENT "1" --> "*" SUBSCR_PARAMETER
ALERT_PACKET_CONFIG "1" --> "0" SUBSCRIPTION_SERVICE

```

### SUBSCRIPTION

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

!!! question "Open questions"

      - Replace column `CARD_CONTRACT_REF` with `REFERENCE`?
      - Add new column `TYPE`?
      - Add `UPDATE_DATE` column?
      - Add `ACCOUNT_STATUS` column (to distinguish between inital entries from ONL and "real" subscriptions)?
      - Add column `CREATE_SOURCE` and `UPDATE_SOURCE` to replace `SOURCE`?
      - Add foreign key to `ALERT_COMPANY`?

### SUBSCRIPTION_SERVICE

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

### ALERT_ADDRESS

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

### SUBSCRIPTION_EVENT

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

### SUBSCRIPTION_PARAMETER

| __Column__              | Type(Size)     | Constraint | Description |
| ----------------------- | -------------- | ---------- | ----------- |
| __SUBSCR_PARAMETER_ID__ | NUMBER(9)      | not null   | Sequence |
| ...                     | ...            | ...        | ... |

!!! info

    Out of scope

### ALERT_ACCOUNT_HISTORY and ALERT_ADDRESS_HISTORY

!!! question "Open questions"

    Should we keep the current logic to show the changes of the different tables with "old" and "new" values?

    Or would it be enough have the history tables have the same structure as the "active" table and always insert the whole object tree?
