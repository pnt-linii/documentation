# Setup of development environment for Message Alert (IBO)

Git-Repository: 

  - Source code: <https://gitlab.kazan.priv.atos.fr/sdosmessagealert/sdos-message-alert-ibo.git>
  - Technical documentation: <https://gitlab.kazan.priv.atos.fr/sdosmessagealert/sdos-messagealert-docs.git>


## Software Installation

  - JDK 1.8
  - JBoss 6.4 (standalone installation)
  - Active MQ
  - fakeSMPT (receive emails sent from message-alert-ibo)
  - SMPPSim (receive SMS sent from message-alert-ibo)
  - Eclipse IDE for Enterprise Java Developers (other IDEs should work as well, there are no Eclipse specific files in the git repository) 
  - SoapUI (for test purposes – call message-alert-ibo web services and/or mock IBO webs ervices)
  - H2 Console (web interface for accessing the embedded H2 database)


### JBoss 6.4 Installation

  - Copy and extract JBoss EAP 6.4-zip file from `P:\Source\Team-SDOS\Software\JBoss-EAP-6\jboss-eap-6.4.0.zip`
  - Install JBoss 6.4 patches (`P:\Source\Team-SDOS\Software\JBoss-EAP-6\jboss-eap-6.4.9-patch.zip`, `P:\Source\Team-SDOS\Software\JBoss-EAP-6\jboss-eap-6.4.18-patch.zip`) as described in `P:\Source\Team-SDOS\Software\JBoss-EAP-6\JBOSS EAP 6.4.6\jboss-eap-6.4.0_HowTo.txt`
  - Copy the `conf` global module from the message-alert-ibo git source tree <https://gitlab.kazan.priv.atos.fr/sdosmessagealert/sdos-message-alert-ibo.git> (under `sdos-message-alert-ibo/build\message-alert-ibo/src/main/modules/conf`) into the `${JBOSS_HOME}/modules` directory
  - Copy the `standalone-ma.xml` from <https://gitlab.kazan.priv.atos.fr/sdosmessagealert/sdos-message-alert-ibo.git> (under `sdos-message-alert-ibo/jboss-eap-6.4/standalone/configuration`) into the `${JBOSS_HOME}/standalone/configuration` directory
  - Copy the JBoss resource adapters into the `${JBOSS_HOME}/standalone/deployments` directory:
      - Active MQ resource adapter `activemq-rar-5.13.3.rar` (extract from `S:\Daten\DT\4\09 Teams\B2C\DEV\message-alert\JBoss.zip`)
      - Oracel DB resource adapter `ojdbc6-11.2.0.2.0.jar` and `ojdbc7-12.1.0.2.jar` (extract from `S:\Daten\DT\4\09 Teams\B2C\DEV\message-alert\JBoss.zip`)
      - Websphere MQ resource adapter `wmq.jmsra.rar` (copy from `P:\Source\Team-SDOS\Software\IBM\MQ Series Resource Adapter\wmq.jmsra.rar`)


!!! tip
    **Global `conf` module**: It is also possible to add the module and point to the configuration files in the Git source tree (instead of copying it manually). This has the advantage that changes on the configuration are kept up to date with the current sources.

    Create the folder `${JBOSS_HOME}/modules/conf/main` and add a `module.xml` file to it with the following contents:

    ```xml
    <module xmlns="urn:jboss:module:1.1" name="conf">
      <resources>
        <resource-root path="../../../../../git/sdos-message-alert-ibo/build/message-alert-ibo/src/main/modules/conf/main"/>
      </resources>
    </module>
    ```

    (The `path` should point to the local Git source tree. With JBoss EAP 6.4 the path needs to be relative, absolute paths are not supported.)

### Active MQ Installation
  - Copy and extract zip file `P:\Source\Team-SDOS\Software\ActiveMQBrowser\apache-activemq-5.7.0.fuse-71-047.zip`
  - Start with `activemq.bat`


### fakeSMPT
  - Copy and extract zip file `P:\Source\Team-SDOS\Software\fakeSMPT\fakeSMTP.zip`
  - Start with `java -jar fakeSMTP-2.0.jar`


### SMPPSim
  - Copy and extract zip file `P:\Source\Team-SDOS\Software\SMPPSim\SMPPSim.zip`
  - Start with `startsmppsim.bat`


### SoapUI
  - Install SoapUI (either from `P:\Source\Team-SDOS\Software\SoapUI\SoapUI-x32-5.4.0.exe` or download a newer version)


### H2 Console

  - Download H2 Console war file from <https://github.com/jboss-eap/quickstart/raw/master-eap6/h2-console/h2console.war>
  - Copy it into the JBoss deployment directory `${JBOSS_HOME}/standalone/deployments`
  - After starting JBoss the H2 Console is reachable under <http://localhost:8080/h2console>
  - Connect to JDBC URL `jdbc:h2:mem:sms` with use `sa` and password `sa`

!!! info
    The embedded database will create the Message Alert tables during first connection to the DB from the message-alert-ibo application. Directly after starting JBoss and looking into the H2 Console, the tables will be missing.


## Eclipse Workspace Setup

### Clone Git repository and Import projects

_(Eclipse IDE for Enterprise Java Developers)_

  - Open Git Repositories Perspective
  - In Git Repositories View `Clone a Git repository`
  - In the following modal dialog
      - URI: <https://gitlab.kazan.priv.atos.fr/sdosmessagealert/sdos-message-alert-ibo.git>
      - User & Password: DAS-ID + Password
      - Select Destination Directory
      - Finish
  - Select the cloned repository and `Import Maven projects ...` (requires the `Maven SCM Handler for EGit` Plug-in, if not installed choose `Import Projects ...`)
  - In the following modal dialog it is recommended to select `[groupId].[artifactId]` as Name template under `Advanced` 
  - Let the build finish (it should compile without errors and no local changes)


### Eclipse Server Runtime Environment Setup

!!! tip
    This is optional, it is of course possible to manually deploy the war file to JBoss.

  - Add a new Server Runtime Environment for `Red Hat JBoss Enterprise Application Platform 6.1+ Runtime`
  - In the following `New Server Runtime Environment` dialog
      - Select the JBoss 6.4 installation directory as `Home Directory`
      - Select `JavaSE-1.8` as `Execution Environment`
      - Set `standalone-ma.xml` as `Configuration file`
      - Finish
  - Create a new Server Adapter for the newly created Runtime Environment (default values should be fine)
      - Add the message-alert-ibo module to the server (it is the only module available)
  - Edit the Launch Configuration for the server
      - Under the `Environment` tab add the `JBOSS_HOME` variable with the JBoss 6.4 installation directory as value


### Start development servers

JBoss message-alert-ibo requires that Active MQ and SMPPSim is already running on the configured ports on the local machine (see `standalone-ma.xml` and `wlsi-messageAlertIBO.properties`):

  - Start Active MQ
  - Start SMPPSim
  - Start JBoss with message-alert-ibo.war deployed

!!! tip
    In Eclipse it is possible to define a launch group to simplify startup of Active MQ, fakeSMPT, SMPPSim and JBoss.

