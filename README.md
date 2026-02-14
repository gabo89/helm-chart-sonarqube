SonarQube Helm Chart
=================

- applies to sonarqube community edition 2026.1LTA (latest sonarqube version shown on docs https://docs.sonarsource.com/sonarqube-server/server-installation/installing-the-database)

reusing tis readme to explain more about the managed identity support 

is intended to avoid putting the db password in config either on the file or in any other place icluding aks cluster secret when installing sonarqube

- access token is managed per connecton not per query and this was already done by sonarqube so dont need to worry on connection exhaustion
- acces token for azure and aws is inmediatlely refresh before they expiration
- acces token duration by default last 1 hour for azure and 15 min for aws
- acces token wont interfere with existing  connection, if the acces token expired while the connection is still happenign, the connection wont be close , it will be still open but new connection cant be made 
- connection by itself have also a max duration so they will close eventually handle by the jdbc library itself

managed identity for azure and aws will respect their  chain validation , in this case i have validated wokload identity specifcally for kubernetes cluster for azure and aws but if app is deployed on a vm it will follow the same pattern on token generation, in that case it will ask to metadata endpoint ,

 this pr doesnt close or open any existing compatibility issues for any database, just removed the requirment of password for any task involving db management 

https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html

https://learn.microsoft.com/en-us/azure/developer/java/sdk/authentication/credential-chains

About managed identity support for sonarqube 
-----------

by defalt sonarqube have support for 3 database (each with their own version) during installaton on the db

https://docs.sonarsource.com/sonarqube-server/server-installation/installing-the-database?fallback=true

- PostgreSQL
- Microsoft SQL Server
- Oracle (oracle jdbc by itself doesnt support so all changes are transparent for this )

the pull request doesnt change that but remove the requirement of a password to connect to any of thase as long as the database is located in a place were identities are supported 

for azure some examples are 

postgresql server 
azure sql server 
azure sql managed identity

a sql server installed in a azure virtual machine cannot work since the sql server installed on the vm cannot inherated any permission asigned   to the virtual machine, a similar scenario happens with a sql or posgtesql server installed on a virtual machine but for example a application we have made with managed identity support can be installed and inherated the identity since it will be calling defaultcredential which will trigger the request for token 

validation done in azure
----------------------
the use case i have validated is to deploy the sonarqube in a statefulset as this helm chart provides  with workload identity enabled  which require

- adding the annotation in the service acount
- adding the label to the pod template that allow pod to use workload identity
- adding azure_client_id env variable with client id 
- that will lead the mutator to inject the access token in the pod 

this is shown in the values.yaml at this root level , take it as reference 

```
env:
  - name: AZURE_CLIENT_ID    <<added
    value: 6412d395-f292-4018-a7d3-5d5c5d540825  <<added
...
...
podLabels:
  azure.workload.identity/use: "true" <<added
...
...
serviceAccount:
  create: true
  automountToken: false
  annotations: 
    #the annotation for aws is like eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/YourK8sRole
    azure.workload.identity/client-id: 6412d395-f292-4018-a7d3-5d5c5d540825 <<added
    azure.workload.identity/tenant-id: 18b08011-0a66-4da0-97f0-cc3e9571c9e9 <<added
...
...
jdbcOverwrite:
  enabled: true
  # The JDBC url of the external DB
  jdbcUrl: "jdbc:sqlserver://sonarqube1980.database.windows.net:1433;database=sonar-db;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;"
  #jdbcUrl: jdbc:postgresql://sonarqube-db-pg16.postgres.database.azure.com:5432/sonarqube?sslmode=require
  AzureIdentity: true
  AwsIdentity: false
  # The DB user that should be used for the JDBC connection
  jdbcUsername: "sonarqube-identity"
  #for aws identity, this are required to generate the signature
  AwsRegion: xxx        #region
  AwsHostname: xxx      #db server
  AwsPort: xxxx         #db server port

```

this helm chart have a small modification in jdbc-config.yaml template  to alllow support for new input parameter for managed identity as part of pull request , that is the only change required  

https://github.com/SonarSource/sonarqube/pull/3420

how can i test this feature
-------

the fastest way can be to test locally using the docker image i have built 
```
dockeragent89/local-sonarqube-app:26.2.5-managedidentity
 ```

but companies will require some compliance  , it will better to clone and build the source code from my branch https://github.com/SonarSource/sonarqube/pull/3420 (my branch is gabo89:feature/adding-managed-identity-support)

execute \gradew clean build 

, some zip will be created under sonar-application/build/distributions

paste the zip to official dockerfile from comunity edition 

https://github.com/SonarSource/docker-sonarqube/tree/master/community-build

edit the dockerfile a litle to get the file from the local instead of remote url, after finish use the image in the edited version ofthis helm chart 






