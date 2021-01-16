# Setup of development environment for Message Alert (sempris)

Git-Repository: <https://gitlab.kazan.atosworldline.com/sdosmessagealert/sdos-message-alert>


## Software
  - JDK 1.8
  - JBoss 6.4 (standalone installation) (`P:\Source\Team-SDOS\Software\JBoss-EAP-6\jboss-eap-6.4.0.zip`)
      - JBoss 6.4 patches (`P:\Source\Team-SDOS\Software\JBoss-EAP-6\jboss-eap-6.4.9-patch.zip`, `P:\Source\Team-SDOS\Software\JBoss-EAP-6\jboss-eap-6.4.18-patch.zip`)
  - Active MQ (`P:\Source\Team-SDOS\Software\ActiveMQBrowser\apache-activemq-5.7.0.fuse-71-047.zip`)
  - fakeSMPT (receive emails sent from message-alert - `P:\Source\Team-SDOS\Software\fakeSMPT\fakeSMTP.zip`)
  - SMPPSim (receive SMS sent from message-alert - `P:\Source\Team-SDOS\Software\SMPPSim\SMPPSim.zip`)
  - Eclipse IDE for Enterprise Java Developers (other IDEs should work as well, there are no Eclipse specific files in the git repository) 
  - SoapUI (for test purposes –  call message-alert web services - `P:\Source\Team-SDOS\Software\SoapUI\SoapUI-x32-5.4.0.exe`) 
      - SoapUI example project for message-alert (`S:\Daten\DT\4\09 Teams\B2C\DEV\message-alert\MsgAlertEventService-soapui-project.xml`)


## JBoss 6.4 Installation
  - JBoss 6.4
  - Patches
  - conf module, standalone/configuration, standalone/deployments (activemq, ojdbc6, ojdbc7) (`S:\Daten\DT\4\09 Teams\B2C\DEV\message-alert\JBoss.zip`)


## Active MQ Installation
  - Extract zip file
  - Start with `activemq.bat`


## fakeSMPT
  - Extract zip file 
  - Start with `java -jar fakeSMTP-2.0.jar`


## SMPPSim
  - Extract zip file
  - Start with `startsmppsim.bat`


## Eclipse Workspace Setup

_(Eclipse IDE for Enterprise Java Developers)_

  - Open Git Repositories Perspective
  - In Git Repositories View `Clone a Git repository`
  - In the following modal dialog
      - URI: <https://gitlab.kazan.priv.atos.fr/sdosmessagealert/sdos-message-alert.git>
      - User & Password: DAS-ID + Password
      - Select Destination Directory
      - Finish
  - Select the cloned repository and `Import Maven projects ...` (may require the `Maven SCM Handler for EGit` Plug-in)
  - In the following modal dialog
      - Under `Advanced` select `[groupId].[artifactId]` as Name template (there are two maven modules with the name `message-alert` that would otherwise clash)
      - Finish 
  - Let the build finish (it should compile without errors and no local changes)


## Eclipse Server Runtime Environment Setup

!!! tip
    This is optional, it is of course possible to manually deploy the war file to JBoss.

  - Add a new Server Runtime Environment for `Red Hat JBoss Enterprise Application Platform 6.1+ Runtime`
  - In the following `New Server Runtime Environment` dialog
      - Select the JBoss 6.4 installation directory as `Home Directory`
      - Select `JavaSE-1.8` as `Execution Environment`
      - Set `standalone-ma.xml` as `Configuration file`
      - Finish
  - Create a new Server Adapter for the newly created Runtime Environment (default values should be fine)
      - Add the message-alert module to the server (it is the only module available)
  - Edit the Launch Configuration for the server
      - Under the `Environment` tab add the `JBOSS_HOME` variable with the JBoss 6.4 installation directory as value


## SoapUI
  - Install SoapUI
  - Import or open example project


## Important Configuration Files
  - `<JBoss-Home>\standalone\configuration\standalone-ma.xml`
  - `<JBoss-Home>\modules\conf\main\wlsi-messageAlert.properties`


## Database
  - URI:      `jdbc:oracle:thin:@dbdev:31021:infosert`
  - User:     `SMS`
  - Password: `smsnew`


## Start development servers

JBoss message-alert requires that Active MQ, SMPPSim and fakeSMPT are already running on the configured ports on the local machine (see `standalone-ma.xml` and `wlsi-messageAlert.properties`):

  - Start Active MQ
  - Start SMPPSim
  - Start fakeSMPT
  - Start JBoss with message-alert.war deployed

### Test event-reception endpoint

The event-reception web service endpoint can be tested with SoapUI and the previously imported SoapUI project.

Example payload:

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="http://www.atosworldline.com/wlsi/msg-alert/event/types/1">
   <soapenv:Header/>
   <soapenv:Body>
      <ns:event>
         <ns:header>
            <ns:eventTime>2016-03-17T16:15:20.123</ns:eventTime>
            <!--You have a CHOICE of the next 7 items at this level-->
            <ns:accountId>1000100</ns:accountId>
            <!--
            <ns:webRefCode>?</ns:webRefCode>
            <ns:isAccountId>?</ns:isAccountId>
            <ns:isCardId>?</ns:isCardId>
            <ns:walletRef>?</ns:walletRef>
            <ns:externalId>?</ns:externalId>
            <ns:accountRef>?</ns:accountRef>
            -->
            <ns:cardIdentifier>1234</ns:cardIdentifier>
            <!--<ns:ruleName>45557840_mail_COS_3dsRegister</ns:ruleName>
-->
            <!--<ns:ruleName>4344990_mail_COS_InfoZahlsch</ns:ruleName>-->
            <ns:ruleName>45557840_mail_COS_sspTanOrder</ns:ruleName>
            <!--<ns:ruleName>16950_12096597_SMS_MPW_001</ns:ruleName>-->
            <!--<ns:ruleName>5126251_SMS_MTN_002</ns:ruleName>-->
            <!--1 to 2 repetitions:-->
            <!-- ns:contact type="msisdn">004917277777788</ns:contact -->
            <ns:contact type="email">Qa-ffm-ip@worldline.com</ns:contact>
            <!--Optional:-->
            <!--<ns:mailType>?</ns:mailType>
-->
            <!--Optional:-->
            <!--<ns:txnReference>?</ns:txnReference>
-->
         </ns:header>
         <ns:body>
            <!--You have a CHOICE of the next 2 items at this level-->
            <ns:entry key="authMerchant">My Merchant</ns:entry>
                                 <ns:entry key="mTAN">ABC123</ns:entry>
            <!--
            <ns:encryptedEntry key="?" keyRef="?" keyVer="?" decryptionMethod="?">aeoliam quae</ns:encryptedEntry>
            -->
         </ns:body>
      </ns:event>
   </soapenv:Body>
</soapenv:Envelope>
```


### Test service-subscription-batch endpoint

The subsription batch endpoint can be tested by setting a local folder for `serviceSubscriptionBatchEndpoint` in the `wlsi-messageAlert.properties` file:

```properties
serviceSubscriptionBatchEndpoint=file:///message-alert/subscription-batch-inbox
```

To trigger the batch process, place a file in the configured folder.

Example file:

```
94355126251   ALRT5126251999000166CX+491771902017   X1191119X155645X                                                            X1191119X155645XBASX1191119X155645
94355126251   INST5126251999000166CX+491771902017   X1191119X155645X                                                            X1191119X155645                   
```


### Test service-subscription-online

```xml
 <!-- Voraussetzungen: -->
 <!-- Step-1 querySubscription erfolgreich, Ergebnis ist die AccountId -->
 <!-- Ziel MSISDN bzw EMAIL muss existieren und in QA-Whitelist eingetragen sein -->
 
 <!-- Ereignis AN-melden: erstmalige Angabe von MSISDN oder/und EMAIL --> 
 <!-- Ereignis AB-melden: EMPTY bei MSISDN oder/und EMAIL eingeben -->
 <!-- Ereignis Aktualisieren: anderer Wert bei MSISDN oder/und EMAIL eingeben --> 
 
 <!-- <b> VORSICHT: je BankWorkflow gibt es individuelle Vorschriften ob Info gesendet wird </b> -->
 
 <soapenv:Envelope xmlns:ns="http://www.atosworldline.com/wlsi/msg-alert/subscription/types/2" xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
    <soapenv:Header/>
       <soapenv:Body>
          <ns:setSubscription>

             <!-- HIER die AccountId von querySubscription eintragen -->
             <ns:accountID>1234567</ns:accountID>

             <!-- HIER Telefon-Nr ..... --> 
             <!-- eintragen. Wenn vorher EMPTY, dann AN-meldung  -->
             <!-- eintragen. Wenn vorher andere Nummer, dann Aktualisierung  -->
             <!-- EMPTY, dann AB-meldung  --> 
             <ns:service type="alert">
               <ns:contact type="msisdn">+4915188888888</ns:contact>

               <!-- HIER Email-Adresse ..... --> 
               <!-- eintragen. Wenn vorher EMPTY, dann AN-meldung  -->
               <!-- eintragen. Wenn vorher anderer Wert, dann Aktualisierung  -->
               <!-- EMPTY, dann AB-meldung  -->
               <contact type="email"/>
               <ns:contact type="email">QaTest@worldline.com</ns:contact>
             </ns:service>  

             <!-- Optional -->
             <!-- default-wert = BAS -->
             <ns:packet>
               <ns:name>BAS</ns:name>
             </ns:packet>

          </ns:setSubscription>
       </soapenv:Body>
 </soapenv:Envelope>
```

```json
{
    "cardContractId": 1234567,
    "service": [
      {
        "type": "alert",
        "contact": [
          {
            "type": "msisdn",
            "value: "+4915188888888"
          },
          {
            "type": "email",
            "value: "QaTest@worldline.com"
          }
        ]
      },
      {
        "type": "inst",
        "contact": [
          {
            "type": "msisdn",
            "value: "+4915188888888"
          },
          {
            "type": "email",
            "value: "QaTest@worldline.com"
          }
        ]
      }
    ],
    "packet": "BAS"
}
```