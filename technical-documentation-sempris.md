# Technical Documentation Message Alert (Sempris)

## Camel routes and queues

```plantuml format="svg"


package "event-reception" {
  interface "cxf:bean:eventReceptionEndpoint" as soap_event_reception
  interface "CAS" as cas
  queue "EVENT_RECEPTION" as q_event_reception
  queue "DECRYPTION" as q_decryption
  component "event-reception" as event_reception
  
  soap_event_reception --> event_reception
  event_reception -> q_decryption
  q_decryption -> event_reception
  cas <--> event_reception 
  q_event_reception <- event_reception
  q_event_reception -> event_reception
}

package "alert-processing" {
  queue "ALERT_PROCESSING" as q_alert_processing
  queue "ALERT_DELAY" as q_alert_delay
  queue "ALERT_PROCESSING.SMS_SENT" as q_sms_sent
  queue "ALERT_PROCESSING.EMAIL" as q_email
  queue "ALERT_PROCESSING.EMAIL_SENT" as q_email_sent
  component "alert-processing" as alert_processing
  
  q_alert_delay --> alert_processing
  q_alert_processing --> alert_processing
  alert_processing <-- q_sms_sent
  q_email <- alert_processing
  q_email -> alert_processing
  alert_processing -> q_email_sent
  q_email_sent -> alert_processing
}

package "instalment-offer" {
  queue "INSTALMENT_OFFER" as q_instalment_offer
  component "instalment-offer" as instalment_offer
  
  q_instalment_offer --> instalment_offer
}

package "notification-msg" {
  queue "NOTIFICATION_MSG" as q_notification_msg
  component "notification-msg" as notification_msg
  
  q_notification_msg --> notification_msg
}

package "service-subscription-batch" {
  file "file:///${JBOSS_HOME}/wlsi/ma" as file_service_subscription
  component "WlpFoServiceCallRoute" as wlpfo_service_call_route
  component "WlpFoServiceUpdateRoute" as wlpfo_service_update_route
  component "PreProcessServiceSubscriptionRoute" as pre_procedss_service_subscription_route
  component "PendingRoute" as pending_route
  component "InfoServerCallRoute" as infoserver_call_route
  component "InfoServerUpdaterRoute" as infoserver_updater_route
  component "FileServiceSubscriptionRoute" as file_service_subscription_route
  interface "cxf:bean:serviceSubscriptionBatchWlpfoEndpoint" as soap_subscription_batch_wlpfo_endpoint
  queue "UPDATE_WLPFO_SUBSCRIPTION" as q_update_wlpfo_subscription
  queue "READ_WLPFO_ACCOUNT" as q_read_wlpfo_account
  queue "READ_INFOSERVER_ACCOUNT" as q_read_infoserver_account
  queue "PRE_PROCESS" as q_pre_process
  queue "READ_INFOSERVER_ACCOUNT.PENDING" as q_read_infoserver_account_pending
  queue "READ_WLPFO_ACCOUNT.PENDING" as q_read_wlpfo_account_pending
  queue "UPDATE_INFOSERVER_SUBSCRIPTION" as q_update_infoserver_subscription
  
  q_read_wlpfo_account --> wlpfo_service_call_route
  wlpfo_service_call_route --> soap_subscription_batch_wlpfo_endpoint : read
  wlpfo_service_call_route --> q_read_infoserver_account
  q_update_wlpfo_subscription --> wlpfo_service_update_route
  wlpfo_service_update_route --> soap_subscription_batch_wlpfo_endpoint : update
  q_read_wlpfo_account_pending <-- wlpfo_service_call_route : error
  
  q_pre_process --> pre_procedss_service_subscription_route
  pre_procedss_service_subscription_route --> q_read_wlpfo_account
  
  q_read_infoserver_account_pending --> pending_route
  pending_route -> q_read_infoserver_account
  q_read_wlpfo_account_pending --> pending_route
  pending_route --> q_read_wlpfo_account
  
  q_read_infoserver_account --> infoserver_call_route
  infoserver_call_route --> q_update_infoserver_subscription
  infoserver_call_route -> q_read_infoserver_account_pending : error
  
  q_update_infoserver_subscription --> infoserver_updater_route
  q_update_wlpfo_subscription <-- infoserver_updater_route
  
  file_service_subscription --> file_service_subscription_route
  file_service_subscription_route --> q_pre_process
}

package "service-subscription-online" {
  interface "cxf:bean:serviceSubscriptionOnlineEndpoint" as soap_service_subscription_online
  interface "cxf:bean:serviceSubscriptionOnlineWlpfoEndpoint" as soap_service_wlpfo
  interface "ims-ma://" as ims_service_subscription_online
  component "service-subscription-online" as service_subscription_online
  queue "SSO.UPDATE_IMS" as q_update_ims
  
  soap_service_subscription_online --> service_subscription_online
  soap_service_wlpfo <- service_subscription_online
  service_subscription_online -> q_update_ims
  q_update_ims -> service_subscription_online
  service_subscription_online -> ims_service_subscription_online
  'TODO IMS-Connect, WLPFO
}

package "status-change-event" {
  interface "imsra://" as ims_status_change_event
  component "status-change-event" as status_change_event
  
  ims_status_change_event --> status_change_event
}

package "bonus-points-sms" {
  file "file:///${JBOSS_HOME}/wlsi/ma" as file_bonus_points_sms
  queue "BPS.IDEMPOTENT" as q_bonus_points_sms
  queue "BPS.CONVERTER" as q_bonus_points_sms_converter
  component "bonus-points-sms" as bonus_points_sms
  
  file_bonus_points_sms --> bonus_points_sms
  bonus_points_sms -> q_bonus_points_sms
  q_bonus_points_sms -> bonus_points_sms
  q_bonus_points_sms_converter <- bonus_points_sms
  q_bonus_points_sms_converter -> bonus_points_sms
}

package "atm-posting-batch" {
  file "file:///${JBOSS_HOME}/wlsi/ma" as file_atm_posting_batch
  component "atm-posting-batch" as atm_posting_batch
  queue "APB.IDEMPOTENT" as q_atm_posting_batch_idempotent
  queue "APB.CONVERTER" as q_atm_posting_batch_converter
  
  file_atm_posting_batch --> atm_posting_batch
  atm_posting_batch -> q_atm_posting_batch_idempotent
  q_atm_posting_batch_idempotent -> atm_posting_batch
  q_atm_posting_batch_converter <- atm_posting_batch
  q_atm_posting_batch_converter -> atm_posting_batch
}

package "batch-sms-reader" {
  database "quartz://messageAlertBatch" as db_batch_sms_reader
  component "batch-sms-reader" as batch_sms_reader
  
  db_batch_sms_reader --> batch_sms_reader
}

package "inbound-sms-reader" {
  interface "inboundSmsRouterEndpoint" as inbound_sms_router_endpoint
  component "inbound-sms-router" as inbound_sms_router
  queue "INBOUND_SMS_ROUTER.DELIVERY_RECEIPT" as q_inbound_sms_router_receipt
  queue "INBOUND_SMS_ROUTER.DELIVER_SM" as q_inbound_sms_router_sm
  
  inbound_sms_router_endpoint --> inbound_sms_router
  q_inbound_sms_router_receipt <- inbound_sms_router
  q_inbound_sms_router_receipt -> inbound_sms_router
  inbound_sms_router -> q_inbound_sms_router_sm
  inbound_sms_router <- q_inbound_sms_router_sm
}

package "instalment-acceptance" {
  interface "ims-ma//" as ims_instalment_acceptance
  component "InstalmentAcceptanceValidationRoute" as instalment_acceptance_validation_route
  component "InstalmentAcceptanceReadOfferRoute" as instalment_acceptance_read_offer_route
  component "InstalmentAcceptanceBookCreditPlanRoute" as instalment_acceptance_book_credit_plan_route
  component "InstalmentAcceptanceUpdateOfferStatusRoute" as instalment_acceptance_update_offer_status_route
  component "InstalmentAcceptanceCallAlertProcessingRoute" as instalment_acceptance_call_alert_processing_route
  queue "INSTALMENT_ACCEPTANCE" as q_instalment_acceptance
  queue "INSTALMENT_ACCEPTANCE.READ_OFFER_DATA" as q_instalment_acceptance_read_offer_data
  queue "INSTALMENT_ACCEPTANCE.CONVERT_ALERT_PROCESSING" as q_instalment_acceptance_alert_processing
  queue "INSTALMENT_ACCEPTANCE.BOOK_CREDIT_PLAN" as q_instalment_acceptance_book_credit_plan
  queue "INSTALMENT_ACCEPTANCE.UPDATE_OFFER_STATUS" as q_instalment_acceptance_update_offer_status
  
  q_instalment_acceptance --> instalment_acceptance_validation_route
  instalment_acceptance_validation_route --> q_instalment_acceptance_read_offer_data
  instalment_acceptance_validation_route --> q_instalment_acceptance_alert_processing : error
  q_instalment_acceptance_read_offer_data --> instalment_acceptance_read_offer_route
  instalment_acceptance_read_offer_route --> q_instalment_acceptance_book_credit_plan
  instalment_acceptance_read_offer_route --> q_instalment_acceptance_alert_processing : error
  q_instalment_acceptance_book_credit_plan --> instalment_acceptance_book_credit_plan_route
  instalment_acceptance_book_credit_plan_route --> q_instalment_acceptance_update_offer_status
  ims_instalment_acceptance <-> instalment_acceptance_book_credit_plan_route
  q_instalment_acceptance_update_offer_status --> instalment_acceptance_update_offer_status_route
  instalment_acceptance_update_offer_status_route --> q_instalment_acceptance_alert_processing
  q_instalment_acceptance_alert_processing --> instalment_acceptance_call_alert_processing_route
}

package "sms-provider-message-mobile" {
  component "SmsProviderMessageMobileRoute" as sms_provider_message_mobile_route
  queue "SMS_MM" as q_sms_mm
  
  q_sms_mm -> sms_provider_message_mobile_route
}

package "sms-provider-mobile-view" {
  component "SmsProviderMobileViewRoute" as sms_provider_mobile_view_route
  queue "SMS_MV" as q_sms_mv
  
  q_sms_mv -> sms_provider_mobile_view_route
}

package "sms-provider-sap" {
  component "SmsProviderSapRoute" as sms_provider_sap_route
  queue "SMS_SAP" as q_sms_sap
  
  q_sms_sap -> sms_provider_sap_route
}

cloud "SMS + EMail" as cloud_sms_email {
  interface "smtp://localhost:26" as smtp_endpoint
  interface "smpp://smppclient@localhost:2775" as smpp_mm_client
  interface "smpp://smppclient@localhost:2775" as smpp_mv_client
  interface "smpp://smppclient@localhost:2775" as smpp_sap_client
}

sms_provider_message_mobile_route --> smpp_mm_client
sms_provider_mobile_view_route --> smpp_mv_client
sms_provider_sap_route --> smpp_sap_client

alert_processing --> smtp_endpoint

alert_processing --> q_sms_mm
alert_processing --> q_sms_mv
alert_processing --> q_sms_sap

q_sms_sent <-- sms_provider_message_mobile_route
q_sms_sent <-- sms_provider_mobile_view_route
q_sms_sent <-- sms_provider_sap_route


event_reception --> q_alert_processing
event_reception --> q_alert_delay
event_reception --> q_instalment_offer

instalment_offer --> q_alert_processing

infoserver_updater_route --> q_notification_msg
service_subscription_online --> q_notification_msg

instalment_acceptance_call_alert_processing_route --> q_alert_processing
notification_msg --> q_alert_processing
atm_posting_batch --> q_alert_processing
status_change_event --> q_alert_processing
bonus_points_sms --> q_alert_processing
batch_sms_reader --> q_alert_processing

inbound_sms_router --> q_instalment_acceptance

```

## event-processing

```plantuml format="svg"

box "External Service" #LightGrey
actor user #LightGray
end box

box "Event-Reception" #LightBlue
boundary "cxf:bean:eventReceptionEndpoint" as eventWebservice
queue "EVENT_RECEPTION" as queueEventReception
participant "Event-Reception" as eventReception
queue "DECRYPTION" as queueDecryption
end box

box "CAS" #LightGrey
boundary CAS #LightGray
end box

box "Alert-Processing" #LightYellow
queue "ALERT_PROCESSING" as queueAlertProcessing
queue "ALERT_DELAY" as queueAlertDelayProcessing
end box
box "Instalment-Offer" #LightYellow
queue "INSTALMENT_OFFER" as queueInstalmentOffer
end box

== Event-Reception Webservice ==

activate user #LightGray
user -> eventWebservice
deactivate user

== route:fromEventEndpoint ==

activate eventWebservice
eventWebservice -> eventReception : fromEventEndpoint
deactivate eventWebservice
activate eventReception
eventReception -> queueDecryption
deactivate eventReception

== route:decryptionQueueEndpoint ==

activate queueDecryption
queueDecryption -> eventReception : decryptionQueueEndpoint
deactivate queueDecryption
activate eventReception
eventReception -> eventReception : direct:cas
activate eventReception #DarkSalmon
eventReception -> CAS
activate CAS #DarkSalmon
CAS -> eventReception
deactivate CAS
eventReception -> eventReception
deactivate eventReception
eventReception -> queueEventReception
deactivate eventReception

== route:eventReceptionQueueEndpoint ==

activate queueEventReception
queueEventReception -> eventReception : eventReceptionQueueEndpoint
deactivate queueEventReception
activate eventReception
eventReception -> eventReception
eventReception -> queueAlertProcessing : Alert mit AMQ_SCHEDULED_DELAY == NULL
activate queueAlertProcessing
eventReception -> queueAlertDelayProcessing : Alert mit AMQ_SCHEDULED_DELAY != NULL
activate queueAlertDelayProcessing
eventReception -> queueInstalmentOffer : Instalment
activate queueInstalmentOffer
deactivate eventReception

```

## Notes about event-reception and alert-processing

### Event Reception

  - Decrypt encrypted values from CAS (mTan)
  - Convert currencies: Converts currency codes from ISO-4217 to an alphanumeric representation. The property currencyEntriesProperty contains the fields to convert. For the conversion a infoserver procedure is used (`INFOSERVER.PC_API_MESSAGE.FN_GET_ALPHA_CURR_CODE`).
  - enrichExternalCustomer: Get AlertCustomerId from AlertMasterData
  - enrichAlertAccount:
      - WLP-FO Account Id: Get AlertAccount and map contact data values to EventReceptionDto
         - __TODO__ Does also create a new AlertAccount if none is found! 
      - Web-Reference-Code, Wallet-Reference, Card-Id, Account-Reference: Get Infoserver-Account-Id and set into EventReceptionDto
  - checkMessageType: 
      - Check AddressSource (Alert, Notify, Instalment) of message rule (AlertMasterData) against available contact data for account (msisdn, email)
      - Set agreement or no-agreement based on available contact data
      - Set destination address (msisdn, email): prefers msisdn
  - checkWhiteList: Check destination address against whitelist (if available)
  - __Only Alert__ checkAlertEventQueueing: check alert events if they need to be delayed (__TODO__ analyze condition)
  - Put into next queue (alert, alert-delay, instalment-offer)


### Alert Processing

  - `enrichTemplateAndPropertiesAndReturnNextEndpoint`:
      - `checkRuleForValidPacketAndAuthResult` (can change rule name!)
  - `MessageAlertBusinessFacadeImpl.enrichTemplateAndProperties(AlertProcessingDto)`
      - Sanity checks for `AlertProcessingDto`
      - Fetch `AlertMasterDataDto` for rule name
      - Fetch `AlertTemplatesDto` for `AlertProcessingDto.messageChannel` (template type email or sms) and `AlertMasterDataDto.templateId`
      - Fetch `AlertConfigurationDto` 
          - either by `AlertMasterDataDto.programId` and `serviceType`
          - or by `AlertMasterDataDto.alertCustomerId` and `serviceType`
      - Validate mobilenumber if necessary
  - `AlertProcessingRouteHelper.prepareForVelocityComponent(Exchange)`
      - Fetch includes from `ALERT_TEXT`
      - Template placeholder handling for embedded files
      - Velocity preparation



## Interface Subscription

Query Subscription
PAN from bank request -> call IBO to translate PAN to CARD_CONTRACT_REF

Input:

  - CARD_CONTRACT_REF

Output:

  - SubscriptionDto (account, addresses, packet)

