<!-- Release notes generated using configuration in .github/release.yml at 6.0.3 -->

## What's Changed
### 🐞 Bug Fixes
* [UNIC-1415]  Fix gnmi subscription session cleanup
* Fix IOS XR parsing
* [UNIC-1469] Fix commit no rollback
* Add logs for device-discovery
* [UNIC-1700] Fix problem with reading data that are marked in YANG as deprecated
* [UNIC-896] With-default query parameter fix
* Additional logs for SNMP
* Fix getting protocol value
* [UNIC-1699] Fix datastore synchronization
* [UNIC-1726] Improve Union codec error reporting
* [UNIC-1724] Uniconfig shell Unhide operations for callbacks SET and Request
### ✅ New Features
* [UNIC-1708] Check node connection rpc
* [UNIC-1709] Gnmi HealthCheckService
* [UNIC-1711] Add SnmpHealthCheckService
* [UNIC-1712] Netconf HealthCheckService
### 💡 Improvements
* Add shell user into AUDIT_USER header when creating tx
* [UNIC-1685] OpenApi: Add error container to response schema
* Rename Check-node-connection RPC
* Fix dependency convergence issues
* Embedded Kafka - stable archive URL
* Remove protobuf .proto files from gnmi-proto
* [VHD-389] Fix creating subscription to streams.
* [UNIC-1713] Stream connection notifications
* Cleanup leaking test dependencies from distribution
* [UNIC-1223] Remove duplicate dependency declarations
* [UNIC-1223] Cleanup dependency version declarations
### 🔨 Dependency Upgrades
* build(deps): bump act10ns/slack from 2.0.0 to 2.1.0
* build(deps): bump actions/upload-artifact from 4.3.1 to 4.3.3
* build(deps): bump actions/download-artifact from 4.1.4 to 4.1.7
* build(deps): bump org.apache.maven.plugins:maven-jar-plugin from 3.3.0 to 3.4.1
* build(deps): bump com.github.spotbugs:spotbugs-maven-plugin from 4.8.3.1 to 4.8.4.0
* build(deps): bump com.github.spotbugs:spotbugs-annotations from 4.8.3 to 4.8.4
* build(deps): bump io.zonky.test:embedded-postgres from 2.0.6 to 2.0.7
* build(deps): bump commons-io:commons-io from 2.16.0 to 2.16.1
* build(deps): bump commons-cli:commons-cli from 1.6.0 to 1.7.0
* build(deps): bump org.springframework.boot:spring-boot-dependencies f… …rom 3.1.10 to 3.1.11
* build(deps): bump com.puppycrawl.tools:checkstyle from 10.14.2 to 10.16.0
* build(deps): bump com.google.errorprone:error_prone_core from 2.26.1 to 2.27.0
* build(deps): bump com.google.errorprone:error_prone_annotations from 2.26.1 to 2.27.0
* build(deps): bump org.apache.maven.plugins:maven-shade-plugin from 3.5.2 to 3.5.3
* build(deps): bump org.snmp4j:snmp4j from 3.8.0 to 3.8.2
* build(deps): bump bouncycastle.version from 1.78 to 1.78.1
* Use org.ow2.asm:asm-bom
* build(deps): bump jline.version from 3.25.1 to 3.26.1
* build(deps): bump org.springdoc:springdoc-openapi-starter-webmvc-ui from 2.4.0 to 2.5.0
* build(deps): bump com.github.spotbugs:spotbugs-maven-plugin from 4.8.4.0 to 4.8.5.0
* build(deps): bump com.github.spotbugs:spotbugs-annotations from 4.8.4 to 4.8.5
* build(deps): bump com.google.errorprone:error_prone_core from 2.27.0 to 2.27.1
* build(deps): bump com.google.errorprone:error_prone_annotations from 2.27.0 to 2.27.1
* build(deps-dev): bump org.apache.maven.plugin-tools:maven-plugin-annotations from 3.12.0 to 3.13.0
* build(deps): bump org.apache.maven.plugins:maven-plugin-plugin from 3.12.0 to 3.13.0
* build(deps): bump com.google.guava:guava from 33.1.0-jre to 33.2.0-jre
* build(deps): bump org.checkerframework:checker-qual from 3.42.0 to 3.43.0
### 🔧 Other Changes
* Bump to 6.0.3-SNAPSHOT.
* Remove unused field from ManageNodeTask
* Add missing nodes to UniConfig OpenApi doc
* [UNIC-1717] Fix gnmi failed subscription
* [UNIC-1718] Rollback removing AAA encryption service
* Release 6.0.3
