## Table of Contents
1. [About](#about)
1. [System requirements](#system-requirements)
1. [Build Rendezvous service jar file](#build-rendezvous-service-jar-file)
1. [Trust management](#trust-management)
1. [Generate Keystores](#generate-keystores)
1. [Run Rendezvous service](#run-rendezvous-service)
    * [Rendezvous service settings](#rendezvous-service-settings)
    * [Proxy settings](#proxy-settings)
    * [Run service](#run-service)
1. [Rendezvous Service Docker Deployment](demo/README.md)

### About
This document can be used as a quick guide to build and run the Rendezvous service.
Rendezvous service is a software service that permits the Device to find its current Owner.

### System requirements

* **Linux Ubuntu 18.04**.
* **Java 8**.
* **Redis database** [page][2]
* **Redis via SSL** [page][3] - usage of stunnel to secure redis (optional)

### Build Rendezvous service jar file
* To clean files generated by previous build, execute the following command:
```
$ ./gradlew clean
```

* To build the Rendezvous service jar file, execute the following command (if you are behind a proxy please update gradle.properties file check [page][6]):
```
$ ./gradlew build -Dversion="YOUR_VERSION"
The jar file is stored in the directory ./build/libs/uRendezvousService-YOUR_VERSION.jar
```  

* To display all available commands, execute the following command:
```
$ ./gradlew tasks
```

* To generate java documentation, execute the following command:
```
$ ./gradlew javadoc
The documentation is stored in the directory ./build/docs/javadoc
```

* To generate unit tests metrics, execute the following command:
```
$ ./gradlew jacocoTestReport
The code coverage report is stored in the directory ./build/reports/jacoco/test/html
```

### Trust management
Before you read this section please install and set up your Redis database.

High level information:

The Rendezvous Service is able to trust one or more keys in the Ownership Voucher.
This can be done using a back channel to supply public key hashes to the Rendezvous Service as they are created
by cooperating supply-chain entities.
It is imperative, for security reasons, that these keys are stripped of any associated data that identify the key
holder before they are configured into the Rendezvous Service.
The Rendezvous Service can block the whole Ownership Voucher by adding one of the keys into the blacklist.
The Rendezvous Service can block the whole Ownership Voucher by adding GUID into the blacklist.

**Prepare ownership voucher keys whitelist**:
* Create Redis Hash **OP_KEYS_WHITELIST**
* Calculate SHA256 hash of an ownership voucher public key (upper case, example: "0734FAC43DBE455D531930B6A8E024043356541BFFCC7A250E417EC38E217725")
* Add the calculated hash to the **OP_KEYS_WHITELIST** with the calculated hash as a key and "1" as a value, example:
```
$ cat ./trustmanagement/example_whitelist_publickey.der | openssl dgst -sha256 | awk '/s/{print toupper($2)}'
0734FAC43DBE455D531930B6A8E024043356541BFFCC7A250E417EC38E217725
```

```
$ redis-cli
$ 127.0.0.1:6379> HSET OP_KEYS_WHITELIST "0734FAC43DBE455D531930B6A8E024043356541BFFCC7A250E417EC38E217725" 1
$ (integer) 1
$ 127.0.0.1:6379> HEXISTS OP_KEYS_WHITELIST "0734FAC43DBE455D531930B6A8E024043356541BFFCC7A250E417EC38E217725"
$ (integer) 1 
```

**Prepare ownership voucher keys blacklist**:
* Create Redis Hash **OP_KEYS_BLACKLIST**
* Calculate SHA256 hash of an ownership voucher public key (upper case, example "3055924C4AF1A77FD365C380F9B3CFC40C5F8C79B1EC6492F0D15648E9792CA2")
* Add the calculated hash to the **OP_KEYS_BLACKLIST** with the calculated hash as a key and "1" as a value, example:
```
$ cat ./trustmanagement/example_blacklist_publickey.der | openssl dgst -sha256 | awk '/s/{print toupper($2)}'
```

```
$ redis-cli
$ 127.0.0.1:6379> HSET OP_KEYS_BLACKLIST "3055924C4AF1A77FD365C380F9B3CFC40C5F8C79B1EC6492F0D15648E9792CA2" 1
$ (integer) 1
$ 127.0.0.1:6379> HEXISTS OP_KEYS_BLACKLIST "3055924C4AF1A77FD365C380F9B3CFC40C5F8C79B1EC6492F0D15648E9792CA2"
$ (integer) 1 
```

**Prepare guid blacklist**:
* Create Redis Hash **GUIDS_BLACKLIST**
* Add the guid (upper case, without dashes, example "0000000000000000000000000000FFFF") to the **GUIDS_BLACKLIST** with the guid as a key and "1" as a value, example: 
```
$ redis-cli
$ 127.0.0.1:6379> HSET GUIDS_BLACKLIST "0000000000000000000000000000FFFF" 1
$ (integer) 1
$ 127.0.0.1:6379> HEXISTS GUIDS_BLACKLIST "0000000000000000000000000000FFFF"
$ (integer) 1 
```

### Generate Keystores

Both keystore and truststore are used to store SSL certificates in the Java programming language.

*The examples of keystore and truststore can be found in the directory certs:*
```
keystore - "rendezvous-keystore.jks"
truststore -  "rendezvous-trustedRootCA.jks"
```
Default passwords for both: 123456

To see how to generate keystore and truststore, visit [page][1].

***Important***: 
-	rendezvous-trustedRootCA.jks must contain a Verification service certificate or root CA to enable communication between the Rendezvous service and the Verification service.
-	in case you want to use the secure redis, rendezvous-trustedRootCA.jks must contain a certificate from the configured redis server [page][3]

### Run Rendezvous service

#### Rendezvous service settings
JVM options can be set to configure Rendezvous service:

| Java Option | Description |
| --- | --- |
| **Hosts** | |
| redis.host | Redis database hostname for RV service to connect to - default localhost |
| redis.port | Redis database port for RV service to connect to - default 6379 |
| redis.ssl | Redis traffic through an SSL tunnel with stunnel - default false |
| server.port | Rendezvous Service host port for HTTPS - default 8000. To configure HTTP port (default: 8001) for the service, update application.properties|
| rendezvous.verificationServiceHost | Verification service host address (https://verify.epid-sbx.trustedservices.intel.com or https://verify.epid.trustedservices.intel.com) |
| **Keystores** | You can use default keystore and trustore or you can generate your own, please review section [keystores](#generate-keystores) |
| javax.net.ssl.trustStore | truststore file - default rendezvous-keystore.jks|
| javax.net.ssl.trustStorePassword | truststore password - default password: 123456|
| server.ssl.key-store | keystore file - default rendezvous-trustedRootCA.jks|
| server.ssl.key-store-password | keystore password - default password: 123456|
| **Modes** | |
| rendezvous.signatureVerification | If set to **false** Intel EPID and ECDSA signatures will **not** be verified |
| rendezvous.opKeyVerification | If set to **false** the whole Ownership Voucher will **not** be verified |
| **Miscellaneous**| |
| rendezvous.tOTokenExpirationTime | The expiration time for JWT counted in minutes |
| rendezvous.ownershipVoucherMaxEntries | It is the maximum number of entries accepted in the Ownership Voucher |
| rendezvous.waitSecondsLimit | Upper bound in seconds. It is the time the Internet location is waiting for a Device to connect |
| spring.profiles.active | Rendezvous service spring profile (production, development, onprem) |
| rendezvous.hmacSecret | The BASE64-encoded algorithm-specific signing key to use to digitally sign the JWT |

#### Proxy settings

* To use external Verifiaction service from behind proxy set the following JVM flags, more info [here][4]:
```
https.proxyPort
https.proxyHost
http.proxyHost
http.proxyHost
```

#### Run service

To run the Rendezvous service as a jar you can use the prepared script rendezvousService.sh.
```
$ ./rendezvousService.sh
```
Or you can manually run Rendezvous service, more info [here][5]:
```
$ java [options] -jar ./build/libs/uRendezvousService*.jar
```

To check whether the Rendezvous service is working properly run the following command: 
```
$ curl --cacert ./certs/ca.cert.pem https://localhost:8000/mp/113/health/full
```

Expected result:
```
{
    "sdoComponents": {
        "database": {
            "status": "OK"
        },
        "rvService": {
            "status": "OK",
            "version": "9.99.999"
        },
        "verificationService": {
            "status": "OK",
            "version": "1.0.3096"
        }
    },
    "sdoStatus": "OK"
}
```

[1]: https://docs.oracle.com/cd/E19509-01/820-3503/6nf1il6er/index.html
[2]: https://redis.io/topics/quickstart
[3]: https://redislabs.com/blog/stunnel-secure-redis-ssl
[4]: https://docs.oracle.com/javase/8/docs/technotes/guides/net/proxies.html
[5]: https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
[6]: https://docs.gradle.org/current/userguide/build_environment.html#sec:accessing_the_web_via_a_proxy
