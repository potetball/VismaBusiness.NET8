# Visma Business SSL Configuration

## Edit as administrator Visma.BusinessHost Service file
*(usually located @ 'C:\Program Files (x86)\Visma\Business\Visma.BusinessHost.exe.config')*
1. In `<serviceBehaviors>`, add a new behavior:
```xml
<behavior name="vbsQwipsBehavior">
    <serviceCredentials>
    <serviceCertificate findValue="QwipsServiceSystemIntegration" x509FindType="FindBySubjectName" storeLocation="LocalMachine" storeName="My" />
    <userNameAuthentication userNamePasswordValidationMode="Custom" customUserNamePasswordValidatorType="Visma.BusinessServices.AuthenticationManager, Visma.BusinessServices" />
    <windowsAuthentication allowAnonymousLogons="false" />
    </serviceCredentials>
    <serviceAuthorization serviceAuthorizationManagerType="Visma.BusinessServices.AuthorizationManager, Visma.BusinessServices" />
    <serviceMetadata httpsGetEnabled="true" httpGetEnabled="true" />
    <serviceDebug includeExceptionDetailInFaults="true" />
    <dataContractSerializer maxItemsInObjectGraph="1000000000" />
</behavior>
```
2. In `<bindings>` `<wsHttpBinding>`, add new `<binding>`
```xml
<binding name="qwipsAuthenticationBinding" maxReceivedMessageSize="2147483647" closeTimeout="23:59:59" openTimeout="23:59:59" receiveTimeout="23:59:59" sendTimeout="23:59:59">
    <readerQuotas maxDepth="2147483647" maxStringContentLength="2147483647" maxArrayLength="2147483647" maxBytesPerRead="2147483647" maxNameTableCharCount="2147483647" />
    <security mode="Transport">
    <message clientCredentialType="UserName" />
    </security>
</binding>
```
3. In `<system.serviceModel>` `<services>`, comment out existing service and add our own, 
**WARNING: WE KEEP http service not to disrupt any existing integrations**, will look like this:
```xml
      <!--
      <service name="Visma.BusinessServices.Generic.GenericService" behaviorConfiguration="vbsSecuredBehavior">
        <endpoint name="CustomAuthenticationEndPoint" address="" binding="wsHttpBinding" bindingConfiguration="customAuthenticationBinding" contract="Visma.BusinessServices.Generic.IGenericService" />
        <endpoint name="WindowsAuthenticationEndPoint" address="Trusted" binding="wsHttpBinding" bindingConfiguration="windowsAuthenticationBinding" contract="Visma.BusinessServices.Generic.IGenericService" />
        <endpoint address="mex" binding="mexHttpBinding" contract="IMetadataExchange" />
        <host>
          <baseAddresses>
            <add baseAddress="http://localhost:2001/GenericService" />
          </baseAddresses>
        </host>
      </service>
      -->
      <service name="Visma.BusinessServices.Generic.GenericService" behaviorConfiguration="vbsQwipsBehavior">
        <endpoint name="CustomAuthenticationEndPoint" address="" binding="wsHttpBinding" bindingConfiguration="customAuthenticationBinding" contract="Visma.BusinessServices.Generic.IGenericService" />
        <endpoint name="QwipsAuthenticationEndPoint" address="https://localhost:20443/GenericService" binding="wsHttpBinding" bindingConfiguration="qwipsAuthenticationBinding" contract="Visma.BusinessServices.Generic.IGenericService" />
        <endpoint address="mex" binding="mexHttpsBinding" contract="IMetadataExchange" />
        <endpoint address="mex" binding="mexHttpBinding" contract="IMetadataExchange" />
        <host>
          <baseAddresses>
            <add baseAddress="http://localhost:2001/GenericService" />
            <add baseAddress="https://localhost:20443/GenericService" />
          </baseAddresses>
        </host>
      </service>
```

# Certificate

# 1. Start powershell and paste in 
*(before you execute, make sure you have a C:\Temp folder to store the public certificate)*
```pwsh
 $certname = "qwips.integration.local"
 $cert = New-SelfSignedCertificate -Subject "CN=$certname" -CertStoreLocation "Cert:\LocalMachine\My" -KeyExportPolicy Exportable -KeySpec Signature -KeyLength 2048 -KeyAlgorithm RSA -HashAlgorithm SHA256 -NotAfter (Get-Date).AddYears(25) -DnsName "integration.qwips.no" -FriendlyName "QwipsServiceSolutions" -Type SSLServerAuthentication
 Export-Certificate -Cert $cert -FilePath "C:\Temp\$certname.cer"
```

## 2. Paste in powershell to get certificate thumbprint
*To allow SSL on port 20443 you need the thumbprint later* 
```pwsh
    Get-ChildItem -Path Cert:\LocalMachine\My | ? { $_.Subject -like "*qwips*" }
```
# Configure Windows HTTPS
*(Lookup httpcfg.exe for older windows ssl over https configuration tool)*

## 3. Allow port for HTTPS:
*(Replace `<thumbprint>` with your newly created certificate thumbprint)*

*(Replace `<username>` with the windows service running Visma BusinessHost service example: mycomputer\qwipsuser)*
```pwsh
netsh http add urlacl url=https://+:20443/GenericService user=<username>
netsh http add sslcert ipport=0.0.0.0:20443 appid='{34f28da5-4e24-4654-9f1d-a84e71846d64}' certhash=<thumbprint>
 ```

## Done. You have now configured the server.

# Client

## 1. Install certificate
- Copy the exported .cer located in the temp folder file to client 
- open windows certificate store
- install certificate in "trusted root certification authorities"

## 2. Edit hosts file
*(located in 'C:\windows\system32\drivers\etc\hosts')*
 - add new line/record into host file:
```
    <ip-address-to-visma-business> integration.qwips.no
```
 - open a web browser and browse: 
    https://integration.qwips.no:20443/GenericService

## Done. You have now configured the client certificate

# .NET 8
## 1. Nuget
- Install Microsoft the Nuget 'System.ServiceModel.Primitivies'

## 2. GenericServiceClient
- Create a WSHttpBinding roughly the same as the legacy `<serviceModel>` definition
```C#
WSHttpBinding binding = new WSHttpBinding
{
    Name = "vbsSecuredBinding",
    CloseTimeout = TimeSpan.FromHours(23).Add(TimeSpan.FromMinutes(59)).Add(TimeSpan.FromSeconds(59)),
    OpenTimeout = TimeSpan.FromHours(23).Add(TimeSpan.FromMinutes(59)).Add(TimeSpan.FromSeconds(59)),
    ReceiveTimeout = TimeSpan.FromHours(23).Add(TimeSpan.FromMinutes(59)).Add(TimeSpan.FromSeconds(59)),
    SendTimeout = TimeSpan.FromHours(23).Add(TimeSpan.FromMinutes(59)).Add(TimeSpan.FromSeconds(59)),
    MaxBufferPoolSize = 524288,
    MaxReceivedMessageSize = 1073741824,
    ReaderQuotas = new XmlDictionaryReaderQuotas
    {
        MaxDepth = 1000000000,
        MaxStringContentLength = 1000000000,
        MaxArrayLength = 1000000000,
        MaxBytesPerRead = 1000000000,
        MaxNameTableCharCount = 1000000000
    },
    Security = new WSHttpSecurity
    {
        Mode = SecurityMode.TransportWithMessageCredential,
        Message = new NonDualMessageSecurityOverHttp
        {
            ClientCredentialType = MessageCredentialType.UserName
        },
    },
};
```
- Create endpoint
```C#
    EndpointAddress endpointAddress = new EndpointAddress(
        new Uri("https://integration.qwips.no/GenericService"),
        new DnsEndpointIdentity("integration.qwips.no")
    );
```
- Instantiate your GenericServiceClient using the arguments
```C#
    serviceClient = new GenericServiceClient(binding, endpointAddress);
```

# UNINSTALLATION
##  To remove the SSL binding
on server: netsh http delete sslcert ipport=0.0.0.0:20443
