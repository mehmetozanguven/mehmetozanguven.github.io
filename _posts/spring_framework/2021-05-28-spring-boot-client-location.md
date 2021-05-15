---
layout: post
title: Find Client's Location In Spring Boot with IP2Location and Update via Scheduling
date: 2021-05-28 14:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this tutorial, **we are going to find location of your clients using spring boot and IP2Location** and also we will look at **how to update IP2Location database periodically with scheduling.**

> **IP2Location** is a non-intrusive IP location lookup technology that retrieves geolocation information with no explicit permission required from users.
>
> Also IP2Location provides free to use geo-ip database.

Topics are:

- [**Github Link**](#github_link)
- [**Free Ip2Location Database**](#free_ip2location_database)
- [**Project Setup**](#project_setup)
  - [**Spring Boot Setup**](#spring_boot_setup)
  - [**IP2Location Java Component**](#ip2location_java_component)
- [**Create an Util Class for Client's Ip address**](#util_class)
- [**Create IP2Location Service**](#iplocation_service)
  - [**Create properties for ip2Location**](#iplocation_properties)
  - [**Read the properties file**](#read_properties_file)
  - [**Create an Ip2LocationService**](#create_iplocation_service)
  - [**Create a method for Ip Look-Up**](#look_up_method)
- [**Create an endpoints**](#create_endpoints)
- [**How to update Ip2Location bin file**](#how_to_update_bin_file)
  - [**Add commons-io dependency**](#add_io_dependency)
  - [**Enable Scheduling**](#enable_scheduling)
  - [**Create Ip2Location Handlers**](#create_ip2location_handlers)
    - [**Create an abstract class for all handlers and create handler object**](#handlers)
    - [**Download the IP2Location Handler**](#download_handler)
    - [**Unzip the new IP2Location**](#unzip_handler)
    - [**Load the new Ip2Location bin file**](#load_new_bin_file)

## Github Link <a name="github_link" />

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-boot-examples/tree/master/spring-ip2Location)

## Free IP2Location Database <a name="free_ip2location_database" />

Go to `https://lite.ip2location.com/` , and create an account. After that Ip2Location gives you a token and corresponding download url.

There are many databases in the Ip2Location. For this tutorial I chose the IP-COUNTRY database. (Database code = DB1.LITE)

After all your download url will be like this:

```wiki
https://www.ip2location.com/download/?token={DOWNLOAD_TOKEN}&file={DATABASE_CODE}
```

## Project Setup <a name="project_setup" />

### Spring Boot Setup <a name="spring_boot_setup" />

Create a spring project with one dependency:

- Spring Web

### IP2Location Java Component <a name="ip2location_java_component" />

Search for ip2location-java component. Right now, latest version is `8.5.2`. Add this dependency to your `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<!-- ... -->
	<properties>
		<java.version>11</java.version>
		<ip2Location.version>8.5.2</ip2Location.version>
	</properties>
	<dependencies>
		<!-- ... -->
		<dependency>
			<groupId>com.ip2location</groupId>
			<artifactId>ip2location-java</artifactId>
			<version>${ip2Location.version}</version>
		</dependency>
        <!-- ... -->
	</dependencies>
<!-- ... -->
</project>
```

## Create an Util Class for Client's Ip address <a name="util_class" />

You can use this util class to find out client's ip address:

> `RequestContextHolder` is class that holds the current web request within the `RequestAttributes` object. You can get the `RequestAttributes` from anywhere in your code.

```java
public class RequestUtils {
    private static final String[] IP_HEADER_CANDIDATES = {
            "X-Forwarded-For",
            "Proxy-Client-IP",
            "WL-Proxy-Client-IP",
            "HTTP_X_FORWARDED_FOR",
            "HTTP_X_FORWARDED",
            "HTTP_X_CLUSTER_CLIENT_IP",
            "HTTP_CLIENT_IP",
            "HTTP_FORWARDED_FOR",
            "HTTP_FORWARDED",
            "HTTP_VIA",
            "REMOTE_ADDR"
    };

    public static String getClientIpAddress() {
        if (RequestContextHolder.getRequestAttributes() == null) {
            return "0.0.0.0";
        }
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();

        for (String header: IP_HEADER_CANDIDATES) {
            String ipList = request.getHeader(header);
            if (ipList != null && ipList.length() != 0 && !"unknown".equalsIgnoreCase(ipList)) {
                return ipList.split(",")[0];
            }
        }
        return request.getRemoteAddr();
    }
}
```

## Create IP2Location Service <a name="iplocation_service" />

First download the ip2location unzip the file.

### Create properties for ip2Location <a name="iplocation_properties" />

Here is the my `application.properties` file:

```properties
app.external.resource.ip2-location.folder=/path/to/ip2Location
app.external.resource.ip2-location.token=yourTokenWillBeHere
app.external.resource.ip2-location.dbName=DB1LITEBIN
```

### Read the properties file <a name="read_properties_file"/>

You can create class for ip2Location properties:

```java
@Configuration
@ConfigurationProperties(prefix = "app.external.resource.ip2-location")
public class Ip2LocationProperties {
    private String folder;
    private String token;
    private String dbName;
    // getters and setters
}
```

### Create an Ip2LocationService <a name="create_iplocation_service"/>

While your service creating, you should read the ipLocation bin file.

```java
import com.ip2location.IP2Location;
// ...
@Service
public class IpLocationServiceImpl {
    public static final String FILE = "/IP2LOCATION-LITE-DB1.BIN";
    private IP2Location ip2Location;

    private final Ip2LocationProperties ip2LocationProperties;
    public Ip2LocationServiceImpl(@Autowired Ip2LocationProperties ip2LocationProperties) {
        this.ip2LocationProperties = ip2LocationProperties;
        openIp2Location();
    }

    private void openIp2Location() {
        String filePath = ip2LocationProperties.getFolder() + FILE;
        this.ip2Location = new IP2Location();
        try {
            this.ip2Location.Open(filePath);
            logger.info("Ip2Location loaded");
        } catch (IOException e) {
            logger.error("Can not open the ip2Location", e);
            throw new RuntimeException("Can not open the ip2Location");
        }
    }
}
```

### Create a method for Ip Look-Up <a name="look_up_method" />

Ip2Location provides easy to use method to look up ip address' location:

> `getCountryLong()` return the country name based on ISO 3166.

```java
@Service
public class IpLocationServiceImpl {
    private static final Logger logger = LoggerFactory.getLogger(IpLocationServiceImpl.class);

    // ...

    public String findIpLocation(String ipAddress) {
        try {
            IPResult ipResult = ip2Location.IPQuery(ipAddress);
            return ipResult.getCountryLong();
        } catch (IOException e) {
            logger.error("Error while ip lookup", e);
            return "undefined";
        }
    }
}
```

## Create an endpoints <a name="create_endpoints" />

I will create two endpoints. One of them is lookup for client's ip location and the other one will find ip location based on the user's input.

```java
@RestController
@RequestMapping("/api")
public class IpLocationController {
    private static final Logger logger = LoggerFactory.getLogger(IpLocationController.class);
    private final MyIpLocation myIpLocation;

    public IpLocationController(@Autowired @Qualifier("ip2Location") MyIpLocation myIpLocation) {
        this.myIpLocation = myIpLocation;
    }

    @GetMapping("/myLocation")
    public String findClientLocation() {
        String clientIpAddress = RequestUtils.getClientIpAddress();
        logger.info("Client ip address: {}",clientIpAddress);
        return myIpLocation.findIpLocation(clientIpAddress);
    }

    @GetMapping("/findLocation/{ipAddress}")
    public String findIpLocation(@PathVariable String ipAddress) {
        return myIpLocation.findIpLocation(ipAddress);
    }
}
```

> When you are sending the request from your machine, you ip address will be localhost (loopback) address, for ipv6 => _0:0:0:0:0:0:0:1_, and for ipv4 => 127.0.0.1. At the end ip2Location result will be:
>
> ```bash
> curl http://localhost:8080/api/myLocation
> - # ip2Location result for localhost is just => `-`
> ```

If you want to test the other endpoint:

```bash
$ curl http://localhost:8080/api/findLocation/1.2.3.4
Australia
```

## How to update Ip2Location bin file <a name="how_to_update_bin_file" />

You should(or must) update ip2Location database periodically. You can do this via Schedule Job in spring boot.

### Add commons-io dependency <a name="add_io_dependency"/>

I will use this dependency to download file from url to destination file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<!-- ... --->
	<properties>
		<java.version>11</java.version>
		<ip2Location.version>8.5.2</ip2Location.version>
		<apache.commons.io>2.8.0</apache.commons.io>
	</properties>
	<dependencies>
<!-- ... --->
		<dependency>
			<groupId>commons-io</groupId>
			<artifactId>commons-io</artifactId>
			<version>${apache.commons.io}</version>
		</dependency>
	</dependencies>
<!-- ... --->
</project>
```

### Enable Scheduling <a name="enable_scheduling" />

Create a configuration class and enable the scheduling

```java
@Configuration
@EnableWebMvc
@EnableScheduling
public class WebConfiguration implements WebMvcConfigurer {
}
```

### Create a Scheduler for IP2Location

My schedule job will run every two mins.

```java
@Component
public class Ip2LocationSchedule {
    private static final Logger logger = LoggerFactory.getLogger(Ip2LocationSchedule.class);
    public static final String EACH_WEEK_ON_FRIDAY_AT_05_30 = "0 30 5 * * 5";
    public static final String EVERY_TWO_MINS = "0 */1 * * * *";

    private final Ip2Location ip2Location;
    private final Ip2LocationProperties ip2LocationProperties;

    public Ip2LocationSchedule(@Autowired Ip2Location ip2Location,
                               @Autowired Ip2LocationProperties ip2LocationProperties) {
        this.ip2Location = ip2Location;
        this.ip2LocationProperties = ip2LocationProperties;
    }

    @Scheduled(cron = EVERY_TWO_MINS)
    public void downloadIp2LocationDatabase() {
        final String methodName = "downloadIp2LocationDatabase";
        logger.info("method: {}, scheduled job is running", methodName);
        logger.info("Trying to download ip2location lite db");
        // THIS PART RELATED TO THE HANDLERS
        Ip2LocationDownloadHandlerObject handlerObject = new Ip2LocationDownloadHandlerObject();

        Ip2LocationDownloadHandler downloadHandler = new DownloadIp2LocationBinFile(ip2LocationProperties);
        Ip2LocationDownloadHandler unzipIp2Location = new UnzipIp2Location(ip2LocationProperties);
        Ip2LocationDownloadHandler loadNewIp2Location = new LoadNewIp2LocationBinFile(ip2LocationProperties, ip2Location);

        // CHAIN THE HANDLERS
        downloadHandler.setNextHandler(unzipIp2Location);
        unzipIp2Location.setNextHandler(loadNewIp2Location);

        // RUN THE FIRST HANDLER
        downloadHandler.handle(handlerObject);
    }
}
```

### Create IP2Location Handlers <a name="create_ip2location_handlers" />

To download and extract the ip2Location databases, I will create handler(s), and each handler represents only one action (for instance downloadHandler only download the ip2Location's database, and pass the request to the nextHandler which is the zipHandler -unzip file the file- etc..)

#### Create an abstract class for all handlers and create handler object <a name="handlers" />

Each handler will extends the abstract handler.

When handler is passing request to the next handler, all operation will be handled by the handlerObject. (All handlers will lookup the handlerObject for specific action(s))

- Abstract Handler;

```java
public abstract class Ip2LocationDownloadHandler {
    private Ip2LocationDownloadHandler nextHandler;
    public abstract void handle(Ip2LocationDownloadHandlerObject handlerObject);
    public Ip2LocationDownloadHandler getNextHandler() {
        return nextHandler;
    }

    public void setNextHandler(Ip2LocationDownloadHandler nextHandler) {
        this.nextHandler = nextHandler;
    }
}
```

- Handler Object:

```java
public class Ip2LocationDownloadHandlerObject {
    private boolean isFileDownloaded;
    private boolean isFileCanBeOpen;

    public Ip2LocationDownloadHandlerObject() {
        this.isFileDownloaded = false;
        this.isFileCanBeOpen = false;
    }
	// getters and setters
}
```

#### Download the IP2Location Handler <a name="download_handler" />

Download the zip file to the new location and update the handlerObject to communicate with other handlers.

> `handlerObject.setFileDownloaded(true);` means that other handlers can check the whether zip file downloaded or not. And other handlers can take action according to the this boolean value

```java
public class DownloadIp2LocationBinFile extends Ip2LocationDownloadHandler {
    private static final String url = "https://www.ip2location.com/download/";

    private final Ip2LocationProperties ip2LocationProperties;
    public DownloadIp2LocationBinFile(Ip2LocationProperties ip2LocationProperties) {
        this.ip2LocationProperties = ip2LocationProperties;
    }

    @Override
    public void handle(Ip2LocationDownloadHandlerObject handlerObject) {
        try {
            logger.info("Ip2 location download url: {}", generateDownloadUrl());
            URL downloadUrl = new URL(generateDownloadUrl());
            File destinationFile = new File(ip2LocationProperties.getFolder() + Ip2LocationServiceImpl.NEW_ZIP_FILE);
            FileUtils.copyURLToFile(downloadUrl, destinationFile);
            handlerObject.setFileDownloaded(true);
        } catch (IOException e) {
            logger.error("Could not download ip2Location bin file");
            handlerObject.setFileDownloaded(false);
        }
        getNextHandler().handle(handlerObject);
    }

    private String generateDownloadUrl() {
//        https://www.ip2location.com/download/?token={DOWNLOAD_TOKEN}&file={DATABASE_CODE}
        return url + "?token=" + ip2LocationProperties.getToken() + "&file=" + ip2LocationProperties.getDbName();
    }
}

```

#### Unzip the new IP2Location <a name="unzip_handler" />

After downloading the ip2Location, you should unzip it.

This handler creates new bin file from the recently downloaded ip2location

```java
public class UnzipIp2Location extends Ip2LocationDownloadHandler {
    private static final Logger logger = LoggerFactory.getLogger(UnzipIp2Location.class);

    private final Ip2LocationProperties ip2LocationProperties;

    public UnzipIp2Location(Ip2LocationProperties ip2LocationProperties) {
        this.ip2LocationProperties = ip2LocationProperties;
    }

    @Override
    public void handle(Ip2LocationDownloadHandlerObject handlerObject) {
        Path zipFile = Path.of(ip2LocationProperties.getFolder() + Ip2LocationServiceImpl.NEW_ZIP_FILE);
        Path newIp2Location = Path.of(ip2LocationProperties.getFolder() + Ip2LocationServiceImpl.NEW_FILE_PATH);
        try {
            ZipInputStream zis = new ZipInputStream(new FileInputStream(zipFile.toFile()));
            ZipEntry zipEntry = zis.getNextEntry();
            while (zipEntry != null) {
                if (zipEntry.getName().contains(Ip2LocationServiceImpl.IP2_LOCATION_FILE_NAME)) {
                    Files.copy(zis, newIp2Location, StandardCopyOption.REPLACE_EXISTING);
                    logger.info("Correctly load new ip2location file");
                }
                zipEntry = zis.getNextEntry();
            }
        } catch (Exception ex) {
            logger.error("Exception while unzipping ip2Location.zip", ex);
        }
        getNextHandler().handle(handlerObject);
    }
}

```

#### Load the new Ip2Location bin file <a name="load_new_bin_file" />

That's the last handler. It just only copies the new bin file to the existing one and reload the `ip2Location`

```java
public class LoadNewIp2LocationBinFile extends Ip2LocationDownloadHandler {
    private static final Logger logger = LoggerFactory.getLogger(LoadNewIp2LocationBinFile.class);

    private final Ip2LocationProperties ip2LocationProperties;
    private final Ip2LocationService ip2LocationService;

    public LoadNewIp2LocationBinFile(Ip2LocationProperties ip2LocationProperties, Ip2LocationService ip2LocationService) {
        this.ip2LocationProperties = ip2LocationProperties;
        this.ip2LocationService = ip2LocationService;
    }

    @Override
    public void handle(Ip2LocationDownloadHandlerObject handlerObject) {
        if (handlerObject.isFileCanBeOpen()) {
            File newBinFile = new File(ip2LocationProperties.getFolder() + Ip2LocationServiceImpl.NEW_FILE_PATH);
            File currentBinFile = new File(ip2LocationProperties.getFolder() + Ip2LocationServiceImpl.FILE_PATH);
            try {
                FileUtils.copyFile(newBinFile ,currentBinFile);
                ip2LocationService.reloadIp2LocationBin();
            } catch (IOException e) {
                logger.error("Can not copy the new bin file to the current bin file.", e);
                throw new RuntimeException("Expected error for ip2location handler");
            }
        }
    }
}
```

Last but not least, wait it for the next one.
