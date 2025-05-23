---
title: UI Testing
description: Custom UI testing is an optional feature that enables you to create and automatically run UI tests for your custom applications
exl-id: 3009f8cc-da12-4e55-9bce-b564621966dd
---

# UI Testing {#ui-testing}

>[!CONTEXTUALHELP]
>id="aemcloud_nonbpa_uitesting"
>title="UI Testing"
>abstract="Custom UI testing is an optional feature that enables you to create and automatically run UI tests for your applications. UI tests are Selenium-based tests packaged in a Docker image to allow a wide choice in language and frameworks (such as Java and Maven, Node and WebDriver.io, or any other framework and technology built upon Selenium)."

Custom UI testing is an optional feature that enables you to create and automatically run UI tests for your applications.
 
## Overview {#custom-ui-testing}

AEM provides an integrated suite of [Cloud Manager quality gates](/help/implementing/cloud-manager/custom-code-quality-rules.md) to ensure smooth updates to custom applications. In particular, IT test gates already support the creation and automation of custom tests using AEM APIs.

UI tests are packaged in a Docker image to allow a wide choice in language and frameworks (such as Cypress, Selenium, Java and Maven, and JavaScript). Additionally, a UI tests project can easily be generated by using [the AEM Project Archetype.](https://experienceleague.adobe.com/docs/experience-manager-core-components/using/developing/archetype/overview.html)

Adobe encourages the usage of Cypress, as it offers real-time reloading and automatic waiting, which helps save time and improves productivity during testing. Cypress also provides a simple and intuitive syntax, making it easy to learn and use, even for those who are new to testing.

UI tests are executed as part of a specific quality gate for each Cloud Manager pipeline with a [**Custom UI Testing** step](/help/implementing/cloud-manager/deploy-code.md) in [production pipelines](/help/implementing/cloud-manager/configuring-pipelines/configuring-production-pipelines.md) or optionally [non-production pipelines](/help/implementing/cloud-manager/configuring-pipelines/configuring-non-production-pipelines.md). Any UI tests including regression and new functionalities enable errors to be detected and reported.

Unlike custom functional tests, which are HTTP tests written in Java, UI tests can be a Docker image with tests written in any language, as long as they follow the conventions defined in the section [Building UI Tests](#building-ui-tests).

>[!TIP]
>
>Adobe recommends using Cypress for UI testing, following the code provided in the [AEM Test Samples repository](https://github.com/adobe/aem-test-samples/tree/aem-cloud/ui-cypress). 
> 
>Adobe also provides UI test module examples based on JavaScript with WebdriverIO (see [AEM Project Archetype](https://github.com/adobe/aem-project-archetype/tree/master/src/main/archetype/ui.tests)) and Java with WebDriver (see [AEM Test Samples repository](https://github.com/adobe/aem-test-samples/tree/aem-cloud/ui-selenium-webdriver)). 

## Get Started with UI Tests {#get-started-ui-tests}

This section describes the steps required to set up UI tests for execution in Cloud Manager.

1. Decide on the programming language that you want to use.

    * For Cypress, use the sample code from the [AEM Test Samples repository](https://github.com/adobe/aem-test-samples/tree/aem-cloud/ui-cypress).

    * For JavaScript and WDIO, use the sample code which is automatically generated in the `ui.tests` folder of your Cloud Manager repository.
      
      >[!NOTE]
      >
      >If your repository was created before Cloud Manager automatically created `ui.tests` folders, you may also generate the latest version using the [AEM Project Archetype](https://github.com/adobe/aem-project-archetype/tree/master/src/main/archetype/ui.tests).

    * For Java and WebDriver, use the sample code from the [AEM Test Samples repository](https://github.com/adobe/aem-test-samples/tree/aem-cloud/ui-selenium-webdriver).

    * For other programming languages, see the section [Building UI Tests](#building-ui-tests) in this document to set up the test project.

1. Ensure that UI testing is activated as per the section [Customer Opt-In](#customer-opt-in) in this document.

1. Develop your test cases and [run the tests locally](#run-ui-tests-locally).

1. Commit your code into the Cloud Manager repository and execute a Cloud Manager pipeline.

## Building UI Tests {#building-ui-tests}

A Maven project generates a Docker build context. This Docker build context describes how to create a Docker image containing the UI tests, which Cloud Manager uses to generate a Docker image containing the actual UI tests.

This section describes the steps needed to add a UI tests project to your repository. 

>[!TIP]
>
>The [AEM Project Archetype](https://github.com/adobe/aem-project-archetype) can generate a UI Tests project for you, which is compliant to the following description, if you do not have special requirements for the programming language.

### Generate a Docker Build Context {#generate-docker-build-context}

To generate a Docker build context, you need a Maven module that:

* Produces an archive that contains a `Dockerfile` and every other file necessary to build the Docker image with your tests.
* Tags the archive with the `ui-test-docker-context` classifier.

The simplest way to do this is to configure the [Maven Assembly Plugin](https://maven.apache.org/plugins/maven-assembly-plugin/) to create the Docker build context archive and assign the right classifier to it.

You can build UI tests with different technologies and frameworks, but this section assumes that your project is laid out in a way similar to the following.

```text
├── Dockerfile
├── assembly-ui-test-docker-context.xml
├── pom.xml
├── test-module
│   ├── package.json
│   ├── index.js
│   └── wdio.conf.js
└── wait-for-grid.sh
```

The `pom.xml` file takes care of the Maven build. Add an execution to the Maven Assembly Plugin similar to the following.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptors>
            <descriptor>${project.basedir}/assembly-ui-test-docker-context.xml</descriptor>
        </descriptors>
        <tarLongFileMode>gnu</tarLongFileMode>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

This execution instructs the Maven Assembly Plugin to create an archive based on the instructions contained in `assembly-ui-test-docker-context.xml`, called an **assembly descriptor** in the plugin's jargon. The assembly descriptor lists all the files that must be part of the archive.

```xml
<assembly>
    <id>ui-test-docker-context</id>
    <includeBaseDirectory>false</includeBaseDirectory>
    <formats>
        <format>tar.gz</format>
    </formats>
    <fileSets>
        <fileSet>
            <directory>${basedir}</directory>
            <includes>
                <include>Dockerfile</include>
                <include>wait-for-grid.sh</include>
            </includes>
        </fileSet>
        <fileSet>
            <directory>${basedir}/test-module</directory>
            <excludes>
                <exclude>node/**</exclude>
                <exclude>node_modules/**</exclude>
                <exclude>reports/**</exclude>
            </excludes>
        </fileSet>
    </fileSets>
</assembly>
```

The assembly descriptor instructs the plugin to create an archive of type `.tar.gz` and assigns the `ui-test-docker-context` classifier to it. Moreover, it lists the files that must be included in the archive including the following.

* A `Dockerfile`, mandatory for building the Docker image
* The `wait-for-grid.sh` script, whose purposes are described below
* The actual UI tests, implemented by a Node.js project in the `test-module` folder

The assembly descriptor also excludes some files that might be generated while running the UI tests locally. This guarantees a smaller archive and faster builds.

The archive containing the Docker build context is automatically picked up by Cloud Manager, which will build the Docker image containing your tests during its deployment pipelines. Eventually, Cloud Manager will run the Docker image to execute the UI tests against your application.

The build should produce either zero or one archive. If it produces zero archives, the test step passes by default. If the build produces more than one archive, which archive is selected is non-deterministic.

### Customer Opt-In {#customer-opt-in}

For Cloud Manager to build and execute your UI tests, you must opt into this feature by adding a file to your repository.

* The file name must be `testing.properties`.
* The file contents must be `ui-tests.version=1`.
* The file must be under the maven submodule for UI tests next to the `pom.xml` file of the UI tests submodule.
* The file must be at the root of the built `tar.gz` file.

The UI tests build and executions are skipped if this file is not present.

To include a `testing.properties` file in the build artifact, add an `include` statement in the `assembly-ui-test-docker-context.xml` file.

```xml
[...]
<includes>
    <include>Dockerfile</include>
    <include>wait-for-grid.sh</include>
    <include>testing.properties</include> <!-- opt-in test module in Cloud Manager -->
</includes>
[...]
```

>[!NOTE]
>
>If your project does not include this line, you will need to edit the file to opt into UI testing.
>
>The file may contain a line advising not to edit it. This is due to it being introduced into your project before opt-in UI testing was introduced and clients were not intended to edit the file. This can be safely ignored.

If you are using the samples provided by Adobe:

* For the JavaScript-based `ui.tests` folder generated based from the [AEM Project Archetype](https://github.com/adobe/aem-project-archetype/tree/master/src/main/archetype/ui.tests), you can execute below command to add the required configuration.

  ```shell
  echo "ui-tests.version=1" > testing.properties

  if ! grep -q "testing.properties" "assembly-ui-test-docker-context.xml"; then
    awk -v line='                <include>testing.properties</include>' '/<include>wait-for-grid.sh<\/include>/ { printf "%s\n%s\n", $0, line; next }; 1' assembly-ui-test-docker-context.xml > assembly-ui-test-docker-context.xml.new && mv assembly-ui-test-docker-context.xml.new assembly-ui-test-docker-context.xml
  fi
  ```

* The Cypress and Java Selenium test samples provided by Adobe already do have the opt-in flag set.

## Writing UI Tests {#writing-ui-tests}

This section describes the conventions that the Docker image containing your UI tests must follow. The Docker image is built out of the Docker build context described in the previous section.

### Environment Variables {#environment-variables}

The following environment variables are passed to your Docker image at run time, depending on your framework.

|Variable|Examples|Description|Testing Framework|
|---|---|---|---|
|`SELENIUM_BASE_URL`|`http://my-ip:4444`|The URL of the Selenium server|Selenium only|
|`SELENIUM_BROWSER`|`chrome`|The browser implementation used by the Selenium Server|Selenium only|
|`AEM_AUTHOR_URL`|`http://my-ip:4502/context-path`|The URL of the AEM author instance|All|
|`AEM_AUTHOR_USERNAME`|`admin`|The user name to log in to the AEM author instance|All|
|`AEM_AUTHOR_PASSWORD`|`admin`|The password to log in to the AEM author instance|All|
|`AEM_PUBLISH_URL`|`http://my-ip:4503/context-path`|The URL of the AEM publish instance|All|
|`AEM_PUBLISH_USERNAME`|`admin`|The user name to log in to the AEM publish instance|All|
|`AEM_PUBLISH_PASSWORD`|`admin`|The password  to log in to the AEM publish instance|All|
|`REPORTS_PATH`|`/usr/src/app/reports`|The path where the XML report of the test results must be saved|All|
|`UPLOAD_URL`|`http://upload-host:9090/upload`|The URL where to which the file must be uploaded to make them accessible to the testing framework|All|

The Adobe test samples provide helper functions to access the configuration parameters:

* Cypress: use the standard function `Cypress.env('VARIABLE_NAME')`
* JavaScript: See the [lib/config.js](https://github.com/adobe/aem-project-archetype/blob/develop/src/main/archetype/ui.tests/test-module/lib/config.js) module
* Java: See the [Config](https://github.com/adobe/aem-test-samples/blob/aem-cloud/ui-selenium-webdriver/test-module/src/main/java/com/adobe/cq/cloud/testing/ui/java/ui/tests/lib/Config.java) class

### Generate Test Reports {#generate-test-reports}

The Docker image must generate test reports in the JUnit XML format and save them in the path specified by the environment variable `REPORTS_PATH`. The JUnit XML format is a widely used format for reporting the results of tests. If the Docker image uses Java and Maven, standard test modules such as [Maven Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/) and [Maven Failsafe Plugin](https://maven.apache.org/surefire/maven-failsafe-plugin/) can generate such reports out of the box.

If the Docker image is implemented with other programming languages or test runners, check the documentation for the chosen tools for how to generate JUnit XML reports.

>[!NOTE]
>
>The result of the UI testing step is evaluated only based on the test reports. Ensure that you generate the report accordingly for your test execution.
>
>Use assertions instead of just logging an error to STDERR or returning a non-zero exit code otherwise your deployment pipeline may proceed normally.

### Prerequisites {#prerequisites}

* The tests in Cloud Manager are run using a technical admin user.

>[!NOTE]
>
>For running the functional tests from your local machine, create a user with admin-like permissions to achieve the same behavior.

* The containerized infrastructure that is scoped for functional testing is limited by the following boundaries:

| Type                 | Value | Description                                                           |
|----------------------|-------|-----------------------------------------------------------------------|
| CPU                  | 2.0   | Amount of CPU-time reserved per test execution                        |
| Memory               | 1Gi   | Amount of memory allocated to the test, value in gibibytes            |
| Timeout              | 30m   | The duration after which the test is ended.                 |
| Recommended Duration | 15m   | Adobe recommends writing the tests to not take longer than this time. |

>[!NOTE]
>
> Should you need more resources, create a Customer Care case and describe your use-case; Adobe will review your request and provide appropriate assistance.

## Selenium-specific details

>[!NOTE]
>
>This section only applies when Selenium is the chosen test infrastructure.

### Waiting for Selenium to be Ready {#waiting-for-selenium}

Before the tests start, it's the responsibility of the Docker image to ensure that the Selenium server is up and running. Waiting for the Selenium service is a two-steps process.

1. Read the URL of the Selenium service from the `SELENIUM_BASE_URL` environment variable.
1. Poll at regular interval to the [status endpoint](https://github.com/SeleniumHQ/docker-selenium/#waiting-for-the-grid-to-be-ready) exposed by the Selenium API.

Once the Selenium's status endpoint answers with a positive response, the tests can start.

The Adobe UI test samples handle this with the script `wait-for-grid.sh`, which is executed upon Docker startup and starts the actual test execution only once the grid is ready.

### Capture Screenshots and Videos {#capture-screenshots}

The Docker image may generate additional test output (for example, screenshots or videos) and save them in the path specified by the environment variable `REPORTS_PATH`. Any file found below the `REPORTS_PATH` are included in the test result archive.

The test samples provided by Adobe by default create screenshots for any failed test.

You can use the helper functions to create screenshots through your tests.

* JavaScript: [takeScreenshot command](https://github.com/adobe/aem-project-archetype/blob/develop/src/main/archetype/ui.tests/test-module/lib/commons.js)
* Java: [Commands](https://github.com/adobe/aem-test-samples/blob/aem-cloud/ui-selenium-webdriver/test-module/src/main/java/com/adobe/cq/cloud/testing/ui/java/ui/tests/lib/Commands.java)

If a test result archive is created during a UI test execution, you can download it from Cloud Manager using the `Download Details` button under the [**Custom UI Testing** step](/help/implementing/cloud-manager/deploy-code.md).

### Upload Files {#upload-files}

Tests sometimes must upload files to the application being tested. To keep the deployment of Selenium flexible relative to your tests, it is not possible to directly upload an asset directly to Selenium. Instead, uploading a file requires the following steps.

1. Upload the file at the URL specified by the `UPLOAD_URL` environment variable.
   * The upload must be performed in one POST request with a multipart form.
   * The multipart form must have a single file field.
   * This is equivalent to `curl -X POST ${UPLOAD_URL} -F "data=@file.txt"`.
   * Consult the documentation and libraries of the programming language used in the Docker image to know how to perform such an HTTP request.
   * The Adobe test samples provide helper functions for uploading files:
     * JavaScript: See the [getFileHandleForUpload](https://github.com/adobe/aem-project-archetype/blob/develop/src/main/archetype/ui.tests/test-module/lib/wdio.commands.js) command.
     * Java: See the [FileHandler](https://github.com/adobe/aem-test-samples/blob/aem-cloud/ui-selenium-webdriver/test-module/src/main/java/com/adobe/cq/cloud/testing/ui/java/ui/tests/lib/FileHandler.java) class.
1. If the upload is successful, the request returns a `200 OK` response of type `text/plain`.
   * The content of the response is an opaque file handle.
   * You can use this handle in place of a file path in an `<input>` element to test file uploads in your application.

## Running UI Tests Locally {#run-ui-tests-locally}

Before activating UI tests in a Cloud Manager pipeline, it's recommended to run the UI tests locally against the [AEM as a Cloud Service SDK](/help/implementing/developing/introduction/aem-as-a-cloud-service-sdk.md) or against an actual AEM as a Cloud Service instance.

### Cypress Test Sample {#cypress-sample}

1. Open a shell and navigate to the `ui.tests/test-module` folder in your repository

1. Install Cypress and other prerequisites

   ```shell
   npm install
   ```

1. Set the environment variables required for test execution

   ```shell
   export AEM_AUTHOR_URL=https://author-<program-id>-<environment-id>.adobeaemcloud.com
   export AEM_AUTHOR_USERNAME=<user>
   export AEM_AUTHOR_PASSWORD=<password>
   export AEM_PUBLISH_URL=https://publish-<program-id>-<environment-id>.adobeaemcloud.com
   export AEM_PUBLISH_USERNAME=<user>
   export AEM_PUBLISH_PASSWORD=<password>
   export REPORTS_PATH=target/
   ```

1. Run tests with one of the following commands

   ```shell
   npm test              # Using default Cypress browser
   npm run test-chrome   # Using Google Chrome browser
   npm run test-firefox  # Using Firefox browser
   ```

>[!NOTE]
>
>The log files are stored in the `target/` folder of your repository.
>
>For details, see [AEM Test Samples repository](https://github.com/adobe/aem-test-samples/blob/aem-cloud/ui-cypress/test-module/README.md).

### JavaScript WebdriverIO Test Sample {#javascript-sample}

1. Open a shell and navigate to the `ui.tests` folder in your repository

1. Execute the below command to start the tests using Maven

   ```shell
   mvn verify -Pui-tests-local-execution \
    -DAEM_AUTHOR_URL=https://author-<program-id>-<environment-id>.adobeaemcloud.com \
    -DAEM_AUTHOR_USERNAME=<user> \
    -DAEM_AUTHOR_PASSWORD=<password> \
    -DAEM_PUBLISH_URL=https://publish-<program-id>-<environment-id>.adobeaemcloud.com \
    -DAEM_PUBLISH_USERNAME=<user> \
    -DAEM_PUBLISH_PASSWORD=<password>
   ```

>[!NOTE]
>
>* This starts a standalone selenium instance and executes the tests against it.
>* The log files are stored in the `target/reports` folder of your repository
>* You need to ensure that your machine is running the latest Chrome version as the test downloads the latest release of ChromeDriver automatically for testing.
>
>For details, see [AEM Project Archetype repository](https://github.com/adobe/aem-project-archetype/blob/develop/src/main/archetype/ui.tests/README.md).

### Java Selenium WebDriver Test Sample {#java-sample}

1. Open a shell and navigate to the `ui.tests/test-module` folder in your repository

1. Execute the below commands to start the tests using Maven

   ```shell
   # Start selenium docker image (for x64 CPUs)
   docker run --platform linux/amd64 -d -p 4444:4444 selenium/standalone-chrome-debug:latest
   
   # Start selenium docker image (for ARM CPUs)
   docker run -d -p 4444:4444 seleniarm/standalone-chromium

   # Run the tests using the previously started Selenium instance
   mvn verify -Pui-tests-local-execution -DSELENIUM_BASE_URL=http://<server>:4444
   ```

>[!NOTE]
>
>The log files are stored in the `target/reports` folder of your repository.
>
>For details, see [AEM Test Samples repository](https://github.com/adobe/aem-test-samples/blob/aem-cloud/ui-selenium-webdriver/README.md).
