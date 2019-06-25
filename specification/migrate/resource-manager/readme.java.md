## Java

These settings apply only when `--java` is specified on the command line.
Please also specify `--azure-libraries-for-java-folder=<path to the root directory of your azure-libraries-for-java clone>`.

 ``` yaml $(java)
azure-arm: true
namespace: com.microsoft.azure.management.migrate
license-header: MICROSOFT_MIT_NO_CODEGEN
payload-flattening-threshold: 1
output-folder: $(azure-libraries-for-java-folder)/azure-mgmt-migrate
```

### Java multi-api

``` yaml $(java) && $(multiapi)
batch:
  - tag: package-2018-02
```

### Tag: package-2018-02 and java

These settings apply only when `--tag=package-2018-02 --java` is specified on the command line.
Please also specify `--azure-libraries-for-java=<path to the root directory of your azure-sdk-for-java clone>`.

``` yaml $(tag) == 'package-2018-02' && $(java) && $(multiapi)
java:
  namespace: com.microsoft.azure.management.migrate.v2018_02_02
  output-folder: $(azure-libraries-for-java-folder)/migrate/resource-manager/v2018_02_02
regenerate-manager: true
generate-interface: true
```