# Exercise 24: Make your Application Secure

## Learning Goal
In the previous exercise you learned how to protect your application with the application router. But unauthenticated and/or unauthorized requests could be sent directly to your app - bypassing the application router. Hence, the application itself must also ensure that only those requests are served that are sent from an authenticated and authorized user.

After this exercise you will know how to secure your application and introduce (domain specific) authorization checks.

Your task is to secure your application with the SAP Container Security Library and the Spring Security framework, so that the application blocks all incoming requests if the user is not authenticated or has no authorization for the required scope "$XSAPPNAME.Display".

Note: There is currently no easy way to make a subset of apps 'unreachable' via http(s) from the outside, e.g. by network segregation. But even if we had that capability, it would still be necessary to have authorization checks in the 'backend' for all sensitive operations.

## Prerequisite

Continue with your solution of the last exercise. If this does not work, you can checkout the branch [`solution-23-Setup-Generic-Authorization`](https://github.com/ccjavadev/cc-bulletinboard-ads-spring-webmvc/tree/solution-23-Setup-Generic-Authorization).

## Step 1: Add Maven dependencies

Add the following dependencies to your `pom.xml` using the XML view of Eclipse:

It is sufficient to add the direct dependency on the SAP Container Security Library; The library itself depends on the Spring Security libraries and the transitive dependency on the Spring Security framework will be resolved automatically.

- Add the Spring Security libraries and the `java-container-security` as dependencies into the `pom.xml`: 
```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-core</artifactId>
    <version>4.2.6.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-web</artifactId>
    <version>4.2.6.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-config</artifactId>
    <version>4.2.6.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.sap.xs2.security</groupId>
    <artifactId>java-container-security</artifactId>
    <version>0.30.3</version>
      <exclusions>
        <exclusion>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-core</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-web</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-config</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
- Note: After you've changed the Maven settings, don't forget to update your Eclipse project (`Alt+F5`)!

- Alternatively you can get the current version of the SAP Java Container Security library from [SAP Service Marketplace](https://launchpad.support.sap.com/#/softwarecenter/template/products/%20_APP=00200682500000001943&_EVENT=DISPHIER&HEADER=Y&FUNCTIONBAR=N&EVENT=TREE&NE=NAVIGATE&ENR=73555000100200004333&V=MAINT&TA=ACTUAL&PAGE=SEARCH/XS%20JAVA%201) (the filename currently is XS_JAVA_8-70001362.ZIP. The version/filename may change in the future).


## Step 2: Configure Spring Security

### Add and modify `WebSecurityConfig` class
- Create a `WebSecurityConfig` class in the package `com.sap.bulletinboard.ads.config` and copy the code from [here](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/src/main/java/com/sap/bulletinboard/ads/config/WebSecurityConfig.java).
- Change the value of field `XSAPPNAME` from `"bulletinboard-d012345"` to `"bulletinboard-<Your d/c/i-User>"`.  
**Note:** The value of `private static final String XSAPPNAME` must be equal to the value of `xsappname`, that is defined in your `xs-security.json` file. 


#### Explanation
The responsibility of the `WebSecurityConfig` class is to **__configure__** the Spring Security framework. The class represents a java based configuration, which has been derived from the XML based configuration of the [XSA java-hello-word sample application](https://github.wdf.sap.corp/xs2-samples/java-hello-world/blob/master/java/src/main/webapp/WEB-INF/). You can use this class as a template for your own applications. Change the `antMatchers` method with the respective HTTP method and URL pattern of the service endpoint(s) to be secured. Change the `access` method with the authorization a user requires for that URL pattern. You can add as many `antMatchers(...).access(...)` method chains as you wish.

Technically - under the hood - the default [`AffirmativeBased`](http://docs.spring.io/autorepo/docs/spring-security/current/apidocs/org/springframework/security/access/vote/AffirmativeBased.html) `AccessDecicionManager` is used. This holds the `WebExpressionVoter`, which in turn makes use of the `OAuth2WebSecurityExpressionHandler` that handles Spring EL expressions like `hasRole` or `isAuthenticated` ([read more](https://docs.spring.io/spring-security/site/docs/3.0.x/reference/el-access.html)). If you require more than one Voter you can specify a "custom" `AccessDecicionManager` such as [`UnanimousBased`](http://docs.spring.io/autorepo/docs/spring-security/current/apidocs/org/springframework/security/access/vote/UnanimousBased.html).

### Activate Security by registering `springSecurityFilterChain` Servlet Filter
- In order to **activate** the Spring Security framework you need to add a servlet filter in the `AppInitializer.onStartup()` method.

```java
// register filter with name "springSecurityFilterChain"
servletContext.addFilter(AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME,
                        new DelegatingFilterProxy(AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME))
              .addMappingForUrlPatterns(EnumSet.allOf(DispatcherType.class), false, "/*");
```
The `DelegatingFilterProxy` intercepts the requests and adds a `ServletFilter` chain between the web container and your web application, so that the Spring Security framework can filter out unauthenticated and unauthorized requests.

### Note on how to enable security checks on method level
Now we have enabled security centrally on the web level as defined in `WebSecurityConfig`. Beside of that you have the option to do the authorization checks on method level using [Method Security](http://docs.spring.io/autorepo/docs/spring-security/current/reference/htmlsingle/#jc-method). In this [(spring boot) example](https://github.com/ccjavadev/cc-bulletinboard-ads-spring-boot/commit/c3150c398ba7e18f703dd06e8c5943a261445293) we implement our own expressions by making use of the new [expression-based access control](http://docs.spring.io/spring-security/site/docs/current/reference/html/el-access.html).

## Step 3: Setup Security for Component Tests

The service tests from [Exercise 4](https://github.com/ccjavadev/cc-coursematerial/blob/master/CreateMicroservice/Exercise_4_CreateServiceTests.md) are not affected by the above changes. They are still running even if the configuration in `WebSecurityConfig` class is loaded into the application context. We strongly recommend you to **activate** security for your service level tests to automatically ensure that all of your application endpoints are protected against unauthorized access.  In this step you will learn to "fake" the security infrastructure, so that the Unit Tests can also test the security settings.

### Activate Security
- Similar to our changes in the `AppInitializer.onStartup()` method we also need to make sure, that `springSecurityFilterChain` bean is added as filter to Mock MVC in the `AdvertisementControllerTest` test class:
```java
@Inject //use javax.servlet.Filter
private Filter springSecurityFilterChain;

@Before
public void setUp() throws Exception {
        mockMvc = MockMvcBuilders.webAppContextSetup(context).addFilters(springSecurityFilterChain).build();
}    
```
- Now run your JUnit tests and see them failing because of unexpected `401` ("unauthenticated") status code.

### Fake Test Security Infrastructure
- Create folder `cc-bulletinboard-ads/src/test/resources` and copy the files [privateKey.txt](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/src/test/resources/privateKey.txt), [publicKey.txt](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/src/test/resources/publicKey.txt) into the new folder.

- Copy the implementation of the `TestSecurityConfig` class from [here](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/src/test/java/com/sap/bulletinboard/ads/config/TestSecurityConfig.java) into the **test package** named `com.sap.bulletinboard.ads.config`.  
In productive environments, SAPOfflineTokenServicesCloud reads the public key value from the environment variable `VCAP_SERVICES`. For unit tests, you explicitly set the public key of your test key pair with the `JwtGenerator`. The JwtGenerator takes the public key from the `publicKey.txt` file.

- Copy the implementation of the `JwtGenerator` class from [here](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/src/test/java/com/sap/bulletinboard/ads/testutils/JwtGenerator.java) into a new test package named `com.sap.bulletinboard.ads.testutils`.


## Step 4: Fix and run Component Tests
### Generate a valid JWT Token
- Update the setup of the `AdvertisementControllerTest` test class according to the below code snippet (replace `bulletinboard-d012345` with the XSAPPNAME you defined in step 2):
```java
private String jwt;

@Before
public void setUp() throws Exception {
    ...
    // compute valid token with Display and Update scopes
    // tenant specific XSAPPNAME (appid) looks like <xsappname>!t<tenant specific index> 
    jwt = new JwtGenerator().addScopes( "bulletinboard-d012345!t27.Display", "bulletinboard-d012345!t27.Update").getTokenForAuthorizationHeader();
}
```
Note: The class `JwtGenerator` has the responsibility to generate a JWT Token for the scopes specified via the `addScopes()` method. It returns the token in a format that is suitable for the HTTP `Authorization` header. The generator signs the JWT Token with its private key (taken from file `privateKey.txt`).

### Add `Authorization` header to each HTTP request
The class `AdvertisementControllerTest` must be updated anywhere the test performs a HTTP method call. All HTTP method calls must be updated with an HTTP header field of name `Authorization` and value `jwt`. For example:  

Before... ```get(AdvertisementController.PATH + "/" + id)```  

After...  ```get(AdvertisementController.PATH + "/" + id).header(HttpHeaders.AUTHORIZATION, jwt)```  


### Run JUnit tests
You can now run the JUnit tests as described [in Exercise 4](https://github.com/ccjavadev/cc-coursematerial/blob/master/CreateMicroservice/Exercise_4_CreateServiceTests.md). They should succeed now.


## Step 5: Run and test the service locally

In this step you will prepare the local runtime environment and test your application manually using `Postman` to confirm that your application is now secure.

### Prepare `VCAP_SERVICES`
Based on the `VCAP_SERVICES` environment variable the `spring-security` module instantiates the SecurityContext.

- In Eclipse, open the Tomcat server settings (by double-clicking on the server) and then open the launch configuration. In the Environment tab edit the `VCAP_SERVICES` variable and replace the value with the following:
```javascript
{"postgresql-9.3":[{"name":"postgresql-lite","label":"postgresql-9.3","credentials":{"dbname":"test","hostname":"127.0.0.1","password":"test123!","port":"5432","uri":"postgres://testuser:test123!@localhost:5432/test","username":"testuser"},"tags":["relational","postgresql"],"plan":"free"}],"rabbitmq-lite":[{"credentials":{"hostname":"127.0.0.1","password":"guest","uri":"amqp://guest:guest@127.0.0.1:5672","username":"guest"},"label":"rabbitmq-lite","tags":["rabbitmq33","rabbitmq","amqp"]}],"xsuaa":[{"credentials":{"clientid":"testClient!t27","clientsecret":"dummy-clientsecret","identityzone":"d012345trial","identityzoneid":"a09a3440-1da8-4082-a89c-3cce186a9b6c","tenantid":"d012345trial","tenantmode":"shared","url":"dummy-url","verificationkey":"-----BEGIN PUBLIC KEY-----MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAn5dYHyD/nn/Pl+/W8jNGWHDaNItXqPuEk/hiozcPF+9l3qEgpRZrMx5ya7UjGdvihidGFQ9+efgaaqCLbk+bBsbU5L4WoJK+/t1mgWCiKI0koaAGDsztZsd3Anz4LEi2+NVNdupRq0ScHzweEKzqaa/LgtBi5WwyA5DaD33gbytG9hdFJvggzIN9+DSverHSAtqGUHhwHSU4/mL36xSReyqiKDiVyhf/y6V6eiE0USubTEGaWVUANIteiC+8Ags5UF22QoqMo3ttKnEyFTHpGCXSn+AEO0WMLK1pPavAjPaOyf4cVX8b/PzHsfBPDMK/kNKNEaU5lAXo8dLUbRYquQIDAQAB-----END PUBLIC KEY-----","xsappname":"bulletinboard-d012345"},"label":"xsuaa","name":"uaa-bulletinboard","plan":"application","tags":["xsuaa"]}]}
```
- If you run the application from the command line, update your `localEnvironmentSetup` script accordingly to  [`localEnvironmentSetup.sh`](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/localEnvironmentSetup.sh) ([`localEnvironmentSetup.bat`](https://github.wdf.sap.corp/raw/cc-java/cc-bulletinboard-ads-spring-webmvc/solution-24-Make-App-Secure/localEnvironmentSetup.bat))
- In both cases make sure that you've changed the value of field `xsappname` from "bulletinboard-d012345" to "bulletinboard-<Your d/c/i-User>".

Note: Using this configuration we can mock the XSUAA backing service as we make use of the so-called "offlineToken verification". This way we can produce a valid JWT Token to test our service as described below.

### Generate JWT Token
Before calling the service you need to provide a digitally signed JWT token to simulate that you are an authenticated user. 
- Therefore simply set a breakpoint in `JWTGenerator.getTokenForAuthorizationHeader()` in package `com.sap.bulletinboard.ads.testutils` and run the `JUnit` tests again to fetch the value of `jwt` from there. 

#### Explanation
The generated JWT Token is an "individual one" as it
 - contains specific scope(s) e.g. `bulletinboard-d012345.Display` (as defined in your `WebSecurityConfig` class). Furthermore, note that the scope is composed of **xsappname** e.g. `bulletinboard-d012345` which also needs to be the same as provided as part of the `VCAP_SERVICES`--`xsuaa`--`xsappname`
 - it is signed with the private key which corresponds to the public key we provided as part of the `VCAP_SERVICES`--`xsuaa`--`verificationkey`

### Call local Service 
You can now test the service manually in the browser using the `Postman` chrome plugin. 

- You should get an `401` ("unauthorized") status code for any endpoint (except for `\health`). 
- Add a header field `Authorization` with the value of the **generated JWT token**.
- Now you can check whether you are able to request the `/api/v1/ads` endpoints. In case your offlineToken verification fails, make sure that the `VCAP_SERVICES` environment variable is provided on Tomcat as described above, another restart might be required.


## Step 6: Deploy and test
In this step you are going to deploy your application to Cloud Foundry and discover that you are not any longer authorized to call your service endpoints directly. This is due to to fact that the necessary scopes are not (yet) assigned to your user account. Unlike the previous steps, your application is now running in a productive security environment which enforces the currently existing security policy.

### Bind UAA Service to your application
Before deploying your application to Cloud Foundry you need to bind your application to the UAA service. 
- As part of the `manifest.yml` you need to enhance the list of services bound to the `bulletinboard-ads` application with the name of your XSUAA service:
```
- name: bulletinboard-ads
  services:
  ...
  - uaa-bulletinboard
```
- Now re-deploy your application to Cloud Foundry.

### Call Deployed Service
- Call your service endpoints e.g. GET `https://bulletinboard-ads-d012345.cfapps.sap.hana.ondemand.com` manually using the `Postman` Chrome plugin. You should get an `401` ("unauthorized") status code for any endpoint (except for `\health`). 
- On Cloud Foundry it is not possible to provide a valid JWT token which is accepted by the XSUAA. Therefore, if you would like to provoke a `403` ("forbidden", "insufficient_scope") status code **you need to call your application via the `approuter`** e.g. GET `https://d012345trial-approuter-d012345.cfapps.sap.hana.ondemand.com/ads/api/v1/ads` in order to authenticate yourself and to create a JWT Token with no scopes. **BUT** you probably will get the login screen in HTML as the response. That's why you need to
  - enable the `Interceptor` within `Postman`.  You might need to install another [`Postman Interceptor` Chrome Plugin](https://chrome.google.com/webstore/detail/postman-interceptor/aicmkgpgakddgnaphhhpliifpcfhicfo), which will help you to send requests which uses browser cookies through the `Postman` app. 
  - logon via `Chrome` Browser first and then
  - back in `Postman` send a GET request to `https://d012345trial-approuter-d012345.cfapps.sap.hana.ondemand.com/ads/api/v1/ads` and
  - make sure that you now get a `403` status code.

> **Note:**  
> By default the application router enables **CSRF protection** for any state-changing HTTP method (e.g. POST, PUT, DELETE).
This is required for compliance with [SEC-223](https://wiki.wdf.sap.corp/wiki/display/PSSEC/SEC-223).
In order to successfully perform a state-changing request you will need to provide a valid `x-csrf-token: <token>` header. You can obtain the `<token>` via a `GET` request with a `x-csrf-token: fetch` header to the application router.


## Step 7: Administrate Authorizations for your Business Application
As of now you've configured your xsuaa service with the application security model ([xs-security.json](https://github.com/ccjavadev/cc-bulletinboard-ads-spring-webmvc/blob/solution-24-Make-App-Secure/security/xs-security.json)). With that, the xsuaa has the knowledge about the role-templates. But you, as a User, still have no permission to access the advertisement endpoints, as the required scopes or roles are not yet assigned to your user. 

To administrate authorizations for your business application, perform the steps of the procedure [HowTo Administrate Authorizations for CF Applications using the SAP CP Cockpit][9].

Subsequently you will need to log-in to your application again, so that the authorities are assigned to the user. You can provoke a login screen by clearing your cache. Now you should have full access to all of your application endpoints.

> **Troubleshooting**
> You can analyze the authorities which are assigned to the current user via `https://d012345trial.authentication.sap.hana.ondemand.com/config?action=who`

## Used Frameworks and Tools
- [XS Application Router](https://github.wdf.sap.corp/xs2/approuter.js/blob/master/README.md)

## Further Reading
- [Spring Security Reference](http://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#abstractsecuritywebapplicationinitializer)
- [Method Security Example](https://github.com/ccjavadev/cc-bulletinboard-ads-spring-boot/commit/c3150c398ba7e18f703dd06e8c5943a261445293)
- [Expression-Based Access Control](https://docs.spring.io/spring-security/site/docs/3.0.x/reference/el-access.html)

[9]: https://jam4.sapjam.com/wiki/show/d2dgJlWR9IpwQsLOCmyJj9?_lightbox=true "SAP CP Cockpit UIs for XSUAA "

## Changes since video recording
- Since the video recording we had to make our application security setup multitenant aware. We use the `d012345trial` PaaS tenant in order to test the deployed microservice. One consequence is that the `WebSecurityConfig` needs to handle the tenant specific `$XSAPPNAME` (or `appid`) which is also part of the scope such as `XSAPPNAME!t<tenant index>.DISPLAY`. XSUAA generates this tenant index to ensure unique appids in order to separate the different states of authorization management (e.g. each customer (SaaS tenant) manages its authorizations differently).


***
<dl>
  <dd>
  <div class="footer">&copy; 2018 SAP SE</div>
  </dd>
</dl>
<hr>
<a href="Exercise_23_SetupGenericAuthorization.md">
  <img align="left" alt="Previous Exercise">
</a>
<a href="/TestStrategy/Exercise_25_Create_SystemTest.md">
  <img align="right" alt="Next Exercise">
</a>
