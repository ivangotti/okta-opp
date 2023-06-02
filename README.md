# okta-opp
On Premise provisioning from Okta to a MSSQL database

A - Prepare Server instance with Microsoft SQL Server + DB / Table ;
 

1 - Create a Windows Server and install MS SQL:

If you have an AWS Account, in EC2, you can go to Launch Instance,

Search an AMI from catalog and select Microsoft Windows Server 2022 with SQL Server 2022 Standard ;

For instance type, select t3.xlarge as it is the minimum requirement for this AMI ;

Otherwise, create the instance your way, and download + install MS SQL ;

2 - Connect to the Server and open Microsoft SQL Server Management Studio ;

3 - On the left-sided bar, click on Connect and connect with your Server credentials ;

4 - Right-click on Databases and click on New Database, give it a name (e.g. dbo used for the documentation) and save ;

5 - Right-click on your database and click on New Query, pass the following code and click on Execute:


CREATE TABLE [dbo].[User](
	UserID varchar(50) NOT NULL,
	Enabled varchar(50) NULL,
	Password varchar(50) NULL,
	FirstName varchar(50) NULL,
	MiddleName varchar(50) NULL,
	LastName varchar(50) NULL,
	Email varchar(50) NULL,
	MobilePhone varchar(50) NULL
)

INSERT INTO [User] (UserID, Enabled, Password, FirstName, MiddleName, LastName, Email, MobilePhone)
VALUES ('julia.doe@topcloud.io', 'true', '********', 'Julia', 'R.', 'Doe', 'julia.doe@topcloud.io', '123-456-789'),
('john.smith@topcloud.io', 'true', '********', 'John', 'W.', 'Smith', 'john.smith@topcloud.io', '789-456-123');
6 - Activate SQL Server Authentication mode:

Right-click on your connection at the top of the left-sided object explorer and click on Properties ;

Go to Security, select SQL Server and Windows Authentication mode ;

Click OK and close prompt;

In Windows search, type Services and open it up ;

Scroll until you find SQL Server (MSSQLSERVER), select it and click on Restart on the left-sided menu ;

7 - Activate sa user and set a password

Back to SQL Server Management Studio, Security / Logins and double-click on user sa ;

In General: Type password as password, confirm password, untick Enforce password policy ;

In Status: Under Login, select Enable ;

Click OK ;

B - Serve the MS SQL DB via https protocol and at a specific DNS (e.g. https://scim.topcloud.io/) ;
SCIM Gateway Setup
1 - Download and install NodeJS ;

2 - Install SCIM gateway (follow “Installation“ instructions) ;

3 - Configure “index.js” file as follow (comment every line except the mssql one) ;


#!/usr/bin/env node

//
// ScimGateway plugin startup
// One or more plugin could be started (must be listening on unique ports)
//
// Could use forman module for running in separate environments
// PM2 module for vertical clustering/loadbalancing among cpu's'
// node-http-proxy for horizontal loadbalancing among hosts (or use nginx)
//

// const loki = require('./lib/plugin-loki')
// const restful = require('./lib/plugin-restful')
// const forwardinc = require('./lib/plugin-forwardinc')
const mssql = require('./lib/plugin-mssql')
// const saphana = require('./lib/plugin-saphana')  // prereq: npm install hdb --save
// const api = require('./lib/plugin-api')
// const azureAD = require('./lib/plugin-azure-ad')
4 - Configure the the file {PATH}/scimgateway/config/plugin-mssql.json setting the following values :


{
  "scimgateway": {
    "port": 443,
    ...
    "scim": {
      "version": "1.1",
      "customSchema": "customSchema.json",
      ...
    },
  ...
  },
  "endpoint": {
    "connection": {
      "server": "127.0.0.1",
      "authentication": {
        "type": "default",
        "options": {
          "userName": "sa",
          "password": "password"
        }
      },
      "options": {
        "instanceName": "",
        "port": 1433,
        "database": "dbo",
        ...
      }
    }
  }
}
Note: We use port 443 here as https requests will be sent by Okta ;

5 - Download the customSchema.json file below and add it in the folder {PATH}/scimgateway/config/schemas/ ;

customSchema.json
06 May 2023, 04:31 AM
 

Note: At this stage, by starting up the gateway via the cmd node index.js, you should be able to hit http://127.0.0.1:443/Users in your browser, passing gwadmin and password as default credentials, and return the 2 entries into your SQL Database. Press CTRL + C in the cmd window to stop the script.

Expose SCIM Gateway to a specific address
1 - Into your registrar, create a A record with the DNS or Subdomain of your choice (e.g. scim.topcloud.io) and type your sever Private IP as value ;

2 - Create a publicly trusted certificate for this domain (e.g. using certbot with this doc or with zerossl) ;

3 - Drop cert.pem and privkey.pem in {PATH}/scimgateway/config/certs/ (pem extension is important) ;

4 - Declare the certificate and the key in the {PATH}/scimgateway/config/certs/plugin-mssql.json (refer below);


...
    "certificate": {
      "key": "private.pem",
      "cert": "certificate.pem",
      "ca": null,
      "pfx": {
        "bundle": null,
        "password": null
      }
    },
...
5 - From the {PATH}, execute the following command node index.js . You can stop the process at any time by pressing Ctrl + C ;

Note: The final trigger message should be “now listening on TLS port 443” ;


6 - In your server browser, type “https://<domainName>/ping” ;

Default login (if prompted): gwadmin

Default password (if prompted): password


>> It should return hello

7 - In your server browser, type “https://<domainName>/ServiceProviderConfig”

Default login (if prompted): gwadmin

Default password (if prompted): password


8 - You can also test the url “https://<domainName>/users” ;

C - Install and configure the Okta On Premise Provisioning Agent ;
1 - Please follow the official deployment guide to download and install the Okta On Premise Provisioning agent ;

NOTE: The OPP Agent will have to decrypt information returned by the SCIM Server. So, we need to re-download it and import it into the OPP

2 - Download and Install the Java JDK (Windows / x64 Installer) ;

3 - Open a command prompt and type the following commands (eventually replace scimgateway by the name of your application folder like my-scimgateway in the npm deployment guide) (Note: keytool command default password is changeit):


> cd C:\Program Files\Java\jdk-20\bin
> keytool -importcert -file "C:\scimgateway\config\certs\cert.pem" -keystore "C:\Program Files\Okta\On-Premises Provisioning Agent\current\jre\lib\security\cacerts"
D - Configure the application in Okta ;
 

1 - Go to the Okta Admin console ;

2 - In the header menu, go to Applications / Applications ;

3 - Click on Add Application ;

4 - Click on Create Application ;

5 - In the dialog box, select Web for platform and Secure Web Authentication (SWA) for Sign on method ;

6 - Give a name to your application, and a dummy login url (just type https://www.placeholder.com) and click on Finish ;

7 - Go to the General tab, in App Settings section, click on Edit, select On-Premises Provisioning as Provisioning and Save ;

8 - Go to Provisioning tab and click on Configure SCIM Connector ;

9 - Here below is a configuration example:


10 - Click on the Test Connector Configuration and the following dialog box should appear:


11 - Click on Save

12 - Go to Provisioning tab, and from the left-side bar, configure the To App and To Okta options ;

13 - Go to Import tab, and click on Import Now. Users from the on-prem SQL table should appear ;

Note: 

If the request is long or times out, don’t hesitate to update to Timeout for API Call in Provisioning tab, Integration panel ;

For some reasons, linked to the difference of data structure sent by Okta and expected by the SCIM Gateway, the Okta to App synchronization still needs some additional configuration ;

14 - In the server, edit the file {PATH}/scimgateway/lib/plugin-mssql.js as following (around lines 50 to 65) :

From:


...
const configDir = path.join(__dirname, '..', 'config')
const configFile = path.join(`${configDir}`, `${pluginName}.json`)
const validScimAttr = [ // array containing scim attributes supported by our plugin code. Empty array - all attrbutes are supported by endpoint
  'userName', // userName is mandatory
  'active', // active is mandatory
  'password',
  'name.givenName',
  'name.middleName',
  'name.familyName',
  // "emails",         // accepts all multivalues for this key
  'emails.work', // accepts multivalues if type value equal work (lowercase)
  // "phoneNumbers",
  'phoneNumbers.work'
]
let config = require(configFile).endpoint
...
To:


...
const configDir = path.join(__dirname, '..', 'config')
const configFile = path.join(`${configDir}`, `${pluginName}.json`)
const validScimAttr = [ // array containing scim attributes supported by our plugin code. Empty array - all attrbutes are supported by endpoint
  'userName', // userName is mandatory
  'active', // active is mandatory
  'password',
  'name.givenName',
  'name.middleName',
  'name.familyName',
  'name.formatted',
  "emails",         // accepts all multivalues for this key
  'emails.work', // accepts multivalues if type value equal work (lowercase)
  "phoneNumbers",
  'phoneNumbers.work'
]
let config = require(configFile).endpoint
...
15 - Restart the SCIM Gateway (in cmd window press CTRL + C and re-execute node index.js) ;

16 - In the Okta Admin console, go to Directory > Profile Editor, and click on Mappings on the side of your SQL Application ;

17 - Go to Okta User to Application tab ;

18 - At email mapping config, dropdown the arrow and select Do not map (result should be as below) ;


19 - Click on Save Mappings and Apply updates now ;

20 - In the left-side bar Applications > Applications, go to your SQL application ;

21 - Go to Provisioning tab, in To App settings ;

22 - Click on Edit, select all checkboxes for:

Create Users ;

Update User Attributes ;

Deactivate Users ;

Sync Password ;

23 - Click on Save ;

24 - That’s all, you can CRUD Users to the MS SQL application and / or import them from it ;
