# GeoMesa Web Data

### Building Instructions

This project contains web services for accessing GeoMesa.

If you wish to build this project separately, you can with maven:

```shell
geomesa> mvn clean install -pl geomesa-web/geomesa-web-data
```

### Installation Instructions

To install geomesa-web-data, extract all jars from ```geomesa-web/geomesa-web-data/target/geomesa-web-data-<version>-install.tar.gz```
into your geoserver lib folder.  You will need to copy `spark-<version>-geomesa-assembly.jar` into the geoserver lib folder, and if not already installed you will need to install the [**geomesa-accumulo-gs-plugin**](../../geomesa-gs-plugin/geomesa-accumulo-geoserver-plugin).

To get the required Spark jar, you may build the geomesa-web-install project using the 'assemble' profile.
Please note GeoMesa does not bundle Spark by default, and that Spark has not been approved for distribution
under the GeoMesa license.

You will need hadoop and slf4j jars that are compatible with your spark install. For a full list of jars
from a working GeoServer instance, refer to [Appendix A](#appendix-a-geoserver-jars). Of note, slf4j needs to be version 1.6.1,
you will need to add the hadoop-yarn jars and commons-cli jar.

In addition to installing jars, you will need to ensure that the hadoop configuration files are available
on the classpath. See the next sections for details.

#### Tomcat Installation

In Tomcat, this can be done by editing ```tomcat/bin/setenv.sh```. The exact line will depend
on your environment, but it will likely be one of the following:

```bash
CLASSPATH="$HADOOP_HOME/conf"
```
or
```bash
CLASSPATH="$HADOOP_CONF_DIR"
```

#### Jboss Installation

In Jboss, the easiest way to get the hadoop files on the classpath is to copy the contents of HADDOP_HOME/conf
into the exploded GeoServer war file under WEB-INF/classes.

Alternatively, you can add hadoop as a module, as described here: https://developer.jboss.org/wiki/HowToPutAnExternalFileInTheClasspath

You will need to exclude Jboss' custom slf4j module, as this interferes with Spark. To do so, add the
following exclusion to your GeoServer jboss-deployment-structure.xml:

```xml
<jboss-deployment-structure xmlns="urn:jboss:deployment-structure:1.1">
  <deployment>
    <dependencies>
        ...
    </dependencies>
    <exclusions>
      <module name="org.jboss.logging.jul-to-slf4j-stub" />
    </exclusions>
  </deployment>
</jboss-deployment-structure>

```

#### Advanced Configuration

##### Distributed Jars

The spark context will load a list of distributed jars to the remote cluster. This can be overridden by
supplying a file called ```spark-jars.list``` on the classpath. This file should contain jar file name prefixes,
one per line, which will be loaded from the environment. For example, to load slf4j-api-1.6.1.jar, you
would put a line in the file containing 'slf4j-api'.

##### Distributed Classloading

The spark context may not load distributed jars properly, resulting in serialization exceptions in the remote
cluster. You may force the classloading of distributed jars by setting the following system property:

```bash
-Dorg.locationtech.geomesa.spark.load-classpath=true
```

### Analytic Web Service

The analytic endpoint provides the ability to run spark jobs through a web service.

The main context path is ```/geoserver/geomesa/analytics```

#### Endpoints

The following paths are defined:

* POST /ds/:alias - Register a GeoMesa data store
  * instanceId
  * zookeepers
  * user
  * password
  * tableName
  * auths (optional)
  * visibilities (optional)
  * queryTimeout (optional)
  * queryThreads (optional)
  * recordThreads (optional)
  * writeMemory (optional)
  * writeThreads (optional)
  * collectStats (optional)
  * caching (optional)

  This method must be called to register any data store you wish to query later. It should not be called
  while the spark context is running. Registered data stores will persist between geoserver reboots.

* DELETE /ds/:alias - Delete a previously registered GeoMesa data store

* GET /ds/:alias - Display a registered GeoMesa data store

* GET /ds - Display all registered GeoMesa data stores

* POST /spark/config - Set spark configurations

  Options are passed as parameters. For a list of available options, see:

  https://spark.apache.org/docs/latest/configuration.html#available-properties <br/>
  https://spark.apache.org/docs/latest/running-on-yarn.html#spark-properties <br/>
  http://spark.apache.org/docs/latest/sql-programming-guide.html#caching-data-in-memory <br/>
  http://spark.apache.org/docs/latest/sql-programming-guide.html#other-configuration-options

  Configuration changes will not take place until the Spark SQL context is restarted. Configuration will
  persist between geoserver restarts.

* GET /spark/config - Displays the current spark configurations

* POST /sql/start - Start the Spark SQL context

* POST /sql/stop - Stop the Spark SQL context

* POST /sql/restart - Start, then stop the Spark SQL context

* GET /sql - Run a sql query
  * q or query - the SQL statement to execute
  * splits (optional) - the number of input splits to use in the Spark input format

  This method will execute a SQL query against any registered data stores. The Spark SQL context will be
  started if it is not currently running.

  The 'where' clause of the SQL statement may contain CQL, which will be applied separately. Columns must
  either be namespaced with a simple feature type name, or must be unambiguous among all registered simple
  feature types.

#### Response Formats

Responses can be returned in several different formats. This can be controlled by a request parameter, or
by an Accept header.

Request parameters:

```
format=txt
format=xml
format=json
```

Accept headers:

```
Accept: text/plain
Accept: application/xml
Accept: application/json
```

Text responses to SQL queries will be either TSV or CSV. The delimiter can be controlled with the 'delim'
parameter, which accepts the values 't', 'tab', 'c', or 'comma'.  

#### Example requests

Register a data store:

```
curl -d 'instanceId=myCloud' -d 'zookeepers=zoo1,zoo2,zoo3' -d 'tableName=myCatalog' -d 'user=user' -d 'password=password' http://localhost:8080/geoserver/geomesa/analytics/ds/myCatalog
```

Set the number of executors:

```
curl -d 'spark.executor.instances=10' http://localhost:8080/geoserver/geomesa/analytics/spark/config
```

Group by: 

```
curl --header 'Accept: text/plain' --get --data-urlencode 'q=select mySft.myAttr, count(*) as count from mySft where bbox(mySft.geom, -115, 45, -110, 50) AND mySft.dtg during 2015-03-02T10:00:00.000Z/2015-03-02T11:00:00.000Z group by myattr' http://localhost:8080/geoserver/geomesa/analytics/sql
```

Join:

```
curl --header 'Accept: text/plain' --get --data-urlencode 'q=select mySft.myAttr, myOtherSft.myAttr from mySft, myOtherSft where bbox(mySft.geom, -115, 45, -110, 50) AND mySft.dtg during 2015-03-02T10:00:00.000Z/2015-03-02T11:00:00.000Z AND  bbox(myOtherSft.geom, -115, 45, -110, 50) AND myOtherSft.dtg during 2015-03-02T10:00:00.000Z/2015-03-02T11:00:00.000Z AND mySft.myJoinField = myOtherSft.myJoinField' http://localhost:8080/geoserver/geomesa/analytics/sql
```

### Appendix A: GeoServer Jars

| jar | size |
| --- | ---- |
| accumulo-core-1.5.2.jar | 3748459 |
| accumulo-fate-1.5.2.jar | 99782 |
| accumulo-trace-1.5.2.jar | 116904 |
| activation-1.1.jar | 62983 |
| akka-actor_2.10-2.3.11.jar | 2658152 |
| akka-remote_2.10-2.3.11.jar | 1352129 |
| akka-slf4j_2.10-2.3.11.jar | 15525 |
| algebird-core_2.10-0.9.0.jar | 2712168 |
| aopalliance-1.0.jar | 4467 |
| asm-3.1.jar | 43033 |
| asm-4.0.jar | 46022 |
| avro-1.7.5.jar | 400680 |
| avro-ipc-1.7.7.jar | 192993 |
| avro-ipc-1.7.7-tests.jar | 346580 |
| avro-mapred-1.7.7-hadoop2.jar | 180736 |
| base64-2.3.8.jar | 17008 |
| batik-anim-1.7.jar | 95313 |
| batik-awt-util-1.7.jar | 401858 |
| batik-bridge-1.7.jar | 558892 |
| batik-css-1.7.jar | 310919 |
| batik-dom-1.7.jar | 173530 |
| batik-ext-1.7.jar | 10257 |
| batik-gvt-1.7.jar | 242866 |
| batik-js-1.7.jar | 504741 |
| batik-parser-1.7.jar | 73119 |
| batik-script-1.7.jar | 60604 |
| batik-svg-dom-1.7.jar | 601098 |
| batik-svggen-1.7.jar | 215274 |
| batik-transcoder-1.7.jar | 121997 |
| batik-util-1.7.jar | 128286 |
| batik-xml-1.7.jar | 30843 |
| bcprov-jdk14-1.46.jar | 1824421 |
| bijection-core_2.10-0.7.2.jar | 1805420 |
| cascading-core-2.6.1.jar | 694882 |
| cascading-hadoop-2.6.1.jar | 251670 |
| cascading-local-2.6.1.jar | 43050 |
| cglib-nodep-2.2.jar | 322362 |
| chill_2.10-0.5.2.jar | 221034 |
| chill-hadoop-0.5.2.jar | 4441 |
| chill-java-0.5.2.jar | 47672 |
| common-2.6.0.jar | 211652 |
| commons-beanutils-1.7.0.jar | 188671 |
| commons-cli-1.2.jar	| 41123 |
| commons-codec-1.9.jar | 263965 |
| commons-collections-3.1.jar | 559366 |
| commons-compiler-2.7.8.jar | 30595 |
| commons-compress-1.4.1.jar | 241367 |
| commons-configuration-1.6.jar | 298829 |
| commons-csv-1.0.jar | 34827 |
| commons-dbcp-1.3.jar | 148817 |
| commons-fileupload-1.2.1.jar | 57779 |
| commons-httpclient-3.1.jar | 305001 |
| commons-io-2.1.jar | 163151 |
| commons-jxpath-1.3.jar | 299994 |
| commons-lang-2.6.jar | 284220 |
| commons-lang3-3.3.2.jar | 412739 |
| commons-logging-1.1.1.jar | 60686 |
| commons-math3-3.4.1.jar | 2035066 |
| commons-net-3.3.jar | 280983 |
| commons-pool-1.5.3.jar | 96203 |
| commons-vfs2-2.0.jar | 415578 |
| com.noelios.restlet-1.0.8.jar | 150629 |
| com.noelios.restlet.ext.servlet-1.0.8.jar | 14072 |
| com.noelios.restlet.ext.simple-1.0.8.jar | 10114 |
| compress-lzf-1.0.3.jar | 79845 |
| config-1.2.1.jar | 219554 |
| core-0.26.jar | 342540 |
| curator-client-2.1.0-incubating.jar | 61504 |
| curator-client-2.7.1.jar | 69500 |
| curator-framework-2.7.1.jar | 186273 |
| curator-recipes-2.7.1.jar | 270342 |
| ecore-2.6.1.jar | 1231403 |
| ehcache-1.6.2.jar | 203035 |
| encoder-1.1.jar | 37176 |
| ezmorph-1.0.6.jar | 86487 |
| freemarker-2.3.18.jar | 924269 |
| GeographicLib-Java-1.44.jar | 31693 |
| geomesa-accumulo-datastore-1.2.0-SNAPSHOT.jar | 2338358 |
| geomesa-accumulo-gs-plugin-1.2.0-SNAPSHOT.jar | 388747 |
| geomesa-compute-1.2.0-SNAPSHOT.jar | 122753 |
| geomesa-feature-all-1.2.0-SNAPSHOT.jar | 12349 |
| geomesa-feature-avro-1.2.0-SNAPSHOT.jar | 184531 |
| geomesa-feature-common-1.2.0-SNAPSHOT.jar | 119375 |
| geomesa-feature-kryo-1.2.0-SNAPSHOT.jar | 155137 |
| geomesa-feature-nio-1.2.0-SNAPSHOT.jar | 41752 |
| geomesa-filter-1.2.0-SNAPSHOT.jar | 178627 |
| geomesa-jobs-1.2.0-SNAPSHOT.jar | 543025 |
| geomesa-raster-1.2.0-SNAPSHOT.jar | 165265 |
| geomesa-security-1.2.0-SNAPSHOT.jar | 39567 |
| geomesa-utils-gs-plugin-1.2.0-SNAPSHOT.jar | 979554 |
| geomesa-web-core-1.2.0-SNAPSHOT.jar | 46226 |
| geomesa-web-data-1.2.0-SNAPSHOT.jar | 76904 |
| geomesa-z3-1.2.0-SNAPSHOT.jar | 30148 |
| grizzled-slf4j_2.10-1.0.2.jar | 6418 |
| gs-gwc-2.8.1.jar | 186789 |
| gs-kml-2.8.1.jar | 181462 |
| gs-main-2.8.1.jar | 1815009 |
| gs-ows-2.8.1.jar | 168857 |
| gs-platform-2.8.1.jar | 96235 |
| gs-rest-2.8.1.jar | 60422 |
| gs-restconfig-2.8.1.jar | 244324 |
| gs-sec-jdbc-2.8.1.jar | 57329 |
| gs-sec-ldap-2.8.1.jar | 46250 |
| gs-wcs1_0-2.8.1.jar | 116297 |
| gs-wcs1_1-2.8.1.jar | 143839 |
| gs-wcs2_0-2.8.1.jar | 433067 |
| gs-wcs-2.8.1.jar | 45960 |
| gs-web-core-2.8.1.jar | 1410821 |
| gs-web-demo-2.8.1.jar | 370595 |
| gs-web-gwc-2.8.1.jar | 366214 |
| gs-web-rest-2.8.1.jar | 4291 |
| gs-web-sec-core-2.8.1.jar | 560507 |
| gs-web-sec-jdbc-2.8.1.jar | 23084 |
| gs-web-sec-ldap-2.8.1.jar | 20200 |
| gs-web-wcs-2.8.1.jar | 78046 |
| gs-web-wfs-2.8.1.jar | 36163 |
| gs-web-wms-2.8.1.jar | 142958 |
| gs-web-wps-2.8.1.jar | 148116 |
| gs-wfs-2.8.1.jar | 688301 |
| gs-wms-2.8.1.jar | 923286 |
| gs-wps-core-2.8.1.jar | 396238 |
| gt-api-14.1.jar | 200535 |
| gt-app-schema-resolver-14.1.jar | 14100 |
| gt-arcgrid-14.1.jar | 23790 |
| gt-complex-14.1.jar | 59836 |
| gt-coverage-14.1.jar | 540009 |
| gt-cql-14.1.jar | 197400 |
| gt-data-14.1.jar | 88541 |
| gt-epsg-hsql-14.1.jar | 2330287 |
| gt-geojson-14.1.jar | 63025 |
| gt-geotiff-14.1.jar | 30844 |
| gt-graph-14.1.jar | 170079 |
| gt-grid-14.1.jar | 35097 |
| gt-gtopo30-14.1.jar | 38389 |
| gt-image-14.1.jar | 22163 |
| gt-imageio-ext-gdal-14.1.jar | 81636 |
| gt-imagemosaic-14.1.jar | 394982 |
| gt-jdbc-14.1.jar | 200865 |
| gt-jdbc-postgis-14.1.jar | 49227 |
| gt-main-14.1.jar | 1721596 |
| gt-metadata-14.1.jar | 508938 |
| gt-opengis-14.1.jar | 345854 |
| gt-process-14.1.jar | 58212 |
| gt-process-feature-14.1.jar | 168783 |
| gt-process-geometry-14.1.jar | 12256 |
| gt-process-raster-14.1.jar | 98169 |
| gt-property-14.1.jar | 24413 |
| gt-referencing-14.1.jar | 1171591 |
| gt-render-14.1.jar | 562499 |
| gt-shapefile-14.1.jar | 206371 |
| gt-svg-14.1.jar | 9065 |
| gt-transform-14.1.jar | 40438 |
| gt-wfs-ng-14.1.jar | 243394 |
| gt-wms-14.1.jar | 228458 |
| gt-xml-14.1.jar | 644571 |
| gt-xsd-core-14.1.jar | 310216 |
| gt-xsd-fes-14.1.jar | 69321 |
| gt-xsd-filter-14.1.jar | 105871 |
| gt-xsd-gml2-14.1.jar | 113576 |
| gt-xsd-gml3-14.1.jar | 1548964 |
| gt-xsd-ows-14.1.jar | 121704 |
| gt-xsd-sld-14.1.jar | 175818 |
| gt-xsd-wcs-14.1.jar | 177088 |
| gt-xsd-wfs-14.1.jar | 148421 |
| gt-xsd-wps-14.1.jar | 40339 |
| guava-17.0.jar | 2243036 |
| gwc-core-1.8.0.jar | 554346 |
| gwc-diskquota-core-1.8.0.jar | 91283 |
| gwc-diskquota-jdbc-1.8.0.jar | 53828 |
| gwc-georss-1.8.0.jar | 35732 |
| gwc-gmaps-1.8.0.jar | 8600 |
| gwc-kml-1.8.0.jar | 21946 |
| gwc-rest-1.8.0.jar | 61743 |
| gwc-tms-1.8.0.jar | 10614 |
| gwc-ve-1.8.0.jar | 5478 |
| gwc-wms-1.8.0.jar | 62571 |
| gwc-wmts-1.8.0.jar | 20000 |
| h2-1.1.119.jar | 1207393 |
| hadoop-auth-2.2.0.jar | 49750 |
| hadoop-client-2.2.0.jar | 2559 |
| hadoop-common-2.2.0.jar | 2735584 |
| hadoop-hdfs-2.2.0.jar | 5242252 |
| hadoop-mapreduce-client-app-2.2.0.jar | 482042 |
| hadoop-mapreduce-client-common-2.2.0.jar | 656365 |
| hadoop-mapreduce-client-core-2.2.0.jar | 1455001 |
| hadoop-mapreduce-client-jobclient-2.2.0.jar | 35216 |
| hadoop-mapreduce-client-shuffle-2.2.0.jar | 21537 |
| hadoop-yarn-api-2.2.0.jar	| 1158936
| hadoop-yarn-applications-distributedshell-2.2.0.jar | 32481
| hadoop-yarn-applications-unmanaged-am-launcher-2.2.0.jar | 13300
| hadoop-yarn-client-2.2.0.jar | 94728
| hadoop-yarn-common-2.2.0.jar | 1301627
| hadoop-yarn-server-common-2.2.0.jar | 175554
| hadoop-yarn-server-nodemanager-2.2.0.jar | 467638
| hadoop-yarn-server-resourcemanager-2.2.0.jar | 615387
| hadoop-yarn-server-web-proxy-2.2.0.jar | 25710
| hadoop-yarn-site-2.2.0.jar | 1935
| hsqldb-2.3.0.jar | 1466946 |
| htmlvalidator-1.2.jar | 243854 |
| imageio-ext-arcgrid-1.1.13.jar | 39860 |
| imageio-ext-gdalarcbinarygrid-1.1.13.jar | 5151 |
| imageio-ext-gdal-bindings-1.9.2.jar | 94016 |
| imageio-ext-gdaldted-1.1.13.jar | 4938 |
| imageio-ext-gdalecw-1.1.13.jar | 8106 |
| imageio-ext-gdalecwjp2-1.1.13.jar | 5083 |
| imageio-ext-gdalehdr-1.1.13.jar | 4939 |
| imageio-ext-gdalenvihdr-1.1.13.jar | 5037 |
| imageio-ext-gdalerdasimg-1.1.13.jar | 5033 |
| imageio-ext-gdalframework-1.1.13.jar | 57744 |
| imageio-ext-gdalidrisi-1.1.13.jar | 5006 |
| imageio-ext-gdalkakadujp2-1.1.13.jar | 14978 |
| imageio-ext-gdalmrsid-1.1.13.jar | 8703 |
| imageio-ext-gdalmrsidjp2-1.1.13.jar | 5129 |
| imageio-ext-gdalnitf-1.1.13.jar | 4928 |
| imageio-ext-gdalrpftoc-1.1.13.jar | 4981 |
| imageio-ext-geocore-1.1.13.jar | 26688 |
| imageio-ext-imagereadmt-1.1.13.jar | 26911 |
| imageio-ext-png-1.1.13.jar | 19493 |
| imageio-ext-streams-1.1.13.jar | 52275 |
| imageio-ext-tiff-1.1.13.jar | 335165 |
| imageio-ext-utilities-1.1.13.jar | 40586 |
| itext-2.1.5.jar | 1117661 |
| ivy-2.4.0.jar | 1282424 |
| jackson-annotations-2.4.0.jar | 38605 |
| jackson-core-2.4.4.jar | 225302 |
| jackson-core-asl-1.9.3.jar | 228268 |
| jackson-databind-2.4.4.jar | 1076926 |
| jackson-mapper-asl-1.9.3.jar | 773019 |
| jackson-module-scala_2.10-2.4.4.jar | 549415 |
| jai_codec-1.1.3.jar | 258160 |
| jai_core-1.1.3.jar | 1900631 |
| jai_imageio-1.1.jar | 1140632 |
| janino-2.7.8.jar | 613299 |
| jasypt-1.8.jar | 178961 |
| JavaAPIforKml-2.2.0.jar | 619507 |
| JavaEWAH-0.6.6.jar | 56982 |
| jdom-1.1.3.jar | 151304 |
| jersey-core-1.9.jar | 458739 |
| jersey-server-1.9.jar | 713089 |
| jets3t-0.7.1.jar | 377780 |
| jettison-1.0.1.jar | 56702 |
| jgrapht-core-0.9.0.jar | 333259 |
| jgrapht-ext-0.9.0.jar | 34229 |
| jgridshift-1.0.jar | 11497 |
| joda-convert-1.6.jar | 98818 |
| joda-time-2.3.jar | 581571 |
| json4s-ast_2.10-3.2.10.jar | 83798 |
| json4s-core_2.10-3.2.10.jar | 584691 |
| json4s-jackson_2.10-3.2.10.jar | 39953 |
| json4s-native_2.10-3.2.10.jar | 68747 |
| json-lib-2.2.3-jdk15.jar | 148490 |
| json-simple-1.1.jar | 16046 |
| jsr-275-1.0-beta-2.jar | 91347 |
| jsr305-2.0.3.jar | 33031 |
| jt-affine-1.0.8.jar | 117359 |
| jt-algebra-1.0.8.jar | 62675 |
| jt-attributeop-1.4.0.jar | 3839 |
| jt-bandcombine-1.0.8.jar | 16049 |
| jt-bandmerge-1.0.8.jar | 33910 |
| jt-bandselect-1.0.8.jar | 8474 |
| jt-binarize-1.0.8.jar | 14196 |
| jt-border-1.0.8.jar | 13680 |
| jt-buffer-1.0.8.jar | 18964 |
| jt-classifier-1.0.8.jar | 18998 |
| jt-colorconvert-1.0.8.jar | 64472 |
| jt-colorindexer-1.0.8.jar | 38695 |
| jt-contour-1.4.0.jar | 26019 |
| jt-crop-1.0.8.jar | 9839 |
| jt-errordiffusion-1.0.8.jar | 18398 |
| jt-format-1.0.8.jar | 7110 |
| jt-imagefunction-1.0.8.jar | 12695 |
| jt-iterators-1.0.8.jar | 24168 |
| jt-lookup-1.0.8.jar | 39687 |
| jt-mosaic-1.0.8.jar | 31244 |
| jt-nullop-1.0.8.jar | 7016 |
| jt-orderdither-1.0.8.jar | 25799 |
| jt-piecewise-1.0.8.jar | 36757 |
| jt-rangelookup-1.4.0.jar | 16779 |
| jt-rescale-1.0.8.jar | 19232 |
| jt-rlookup-1.0.8.jar | 18288 |
| jts-1.13.jar | 794991 |
| jt-scale-1.0.8.jar | 89273 |
| jt-stats-1.0.8.jar | 34639 |
| jt-translate-1.0.8.jar | 9978 |
| jt-utilities-1.0.8.jar | 117726 |
| jt-utils-1.4.0.jar | 202267 |
| jt-vectorbin-1.0.8.jar | 19122 |
| jt-vectorbinarize-1.4.0.jar | 10574 |
| jt-vectorize-1.4.0.jar | 14229 |
| jt-warp-1.0.8.jar | 65329 |
| jt-zonal-1.0.8.jar | 32744 |
| jt-zonalstats-1.4.0.jar | 19970 |
| juniversalchardet-1.0.3.jar | 220813 |
| kryo-2.21.jar | 363460 |
| libthrift-0.9.1.jar | 217053 |
| log4j-1.2.14.jar | 367444 |
| lz4-1.3.0.jar | 236880 |
| mail-1.4.jar | 388864 |
| mango-core-1.2.0.jar | 166446 |
| maple-0.13.1.jar | 27673 |
| mesos-0.21.1-shaded-protobuf.jar | 1277883 |
| MetaModel-core-4.3.6.jar | 414462 |
| MetaModel-pojo-4.3.6.jar | 23684 |
| metrics-core-3.1.2.jar | 112558 |
| metrics-graphite-3.1.2.jar | 20852 |
| metrics-json-3.1.2.jar | 15827 |
| metrics-jvm-3.1.2.jar | 39280 |
| mime-util-2.1.3.jar | 119180 |
| minlog-1.2.jar | 4965 |
| net.opengis.fes-14.1.jar | 230225 |
| net.opengis.ows-14.1.jar | 528947 |
| net.opengis.wcs-14.1.jar | 657860 |
| net.opengis.wfs-14.1.jar | 427780 |
| net.opengis.wps-14.1.jar | 196364 |
| netty-3.8.0.Final.jar | 1230201 |
| netty-all-4.0.29.Final.jar | 2054931 |
| objenesis-1.0.jar | 28569 |
| org.json-2.0.jar | 48752 |
| org.restlet-1.0.8.jar | 175177 |
| org.restlet.ext.freemarker-1.0.8.jar | 2180 |
| org.restlet.ext.json-1.0.8.jar | 1730 |
| org.restlet.ext.spring-1.0.8.jar | 4504 |
| org.simpleframework-3.1.3.jar | 225333 |
| org.w3.xlink-14.1.jar | 52959 |
| oro-2.0.8.jar | 65261 |
| paranamer-2.6.jar | 32806 |
| parquet-column-1.7.0.jar | 917052 |
| parquet-common-1.7.0.jar | 21575 |
| parquet-encoding-1.7.0.jar | 285447 |
| parquet-format-2.3.0-incubating.jar | 387188 |
| parquet-generator-1.7.0.jar | 21243 |
| parquet-hadoop-1.7.0.jar | 209622 |
| parquet-jackson-1.7.0.jar | 1048110 |
| picocontainer-1.2.jar | 112635 |
| pngj-2.0.1.jar | 142870 |
| postgresql-8.4-701.jdbc3.jar | 472831 |
| postgresql-9.4-1201-jdbc41.jar | 648487 |
| protobuf-java-2.5.0.jar | 533455 |
| py4j-0.8.2.1.jar | 80850 |
| pyrolite-4.4.jar | 83432 |
| reflectasm-1.07-shaded.jar | 65612 |
| riffle-0.1-dev.jar | 11351 |
| rl_2.10-0.4.10.jar | 226545 |
| RoaringBitmap-0.4.5.jar | 109388 |
| scala-compiler-2.10.5.jar | 14472629 |
| scala-library-2.10.5.jar | 7130772 |
| scalalogging-slf4j_2.10-1.1.0.jar | 79003 |
| scalap-2.10.0.jar | 855012 |
| scala-reflect-2.10.5.jar | 3206179 |
| scalatra_2.10-2.3.0.jar | 1239091 |
| scalatra-auth_2.10-2.3.0.jar | 86192 |
| scalatra-common_2.10-2.3.0.jar | 23201 |
| scalatra-json_2.10-2.3.0.jar | 86792 |
| scalding-args_2.10-0.13.1.jar | 38607 |
| scalding-core_2.10-0.13.1.jar | 3498179 |
| scalding-date_2.10-0.13.1.jar | 111564 |
| serializer-2.7.1.jar | 278281 |
| slf4j-api-1.6.1.jar | 25496 |
| slf4j-log4j12-1.7.10.jar | 8866 |
| snappy-java-1.1.1.7.jar | 594033 |
| spark-catalyst_2.10-1.5.0.jar | 4614818 |
| spark-core_2.10-1.5.0.jar | 11081542 |
| spark-launcher_2.10-1.5.0.jar | 43198 |
| spark-network-common_2.10-1.5.0.jar | 2330017 |
| spark-network-shuffle_2.10-1.5.0.jar | 49264 |
| spark-sql_2.10-1.5.0.jar | 3452234 |
| spark-unsafe_2.10-1.5.0.jar | 43201 |
| spark-yarn_2.10-1.5.0.jar | 520473 |
| spatial4j-0.4.1.jar | 102177 |
| spring-aop-3.1.4.RELEASE.jar | 332932 |
| spring-asm-3.1.4.RELEASE.jar | 53082 |
| spring-beans-3.1.4.RELEASE.jar | 597184 |
| spring-context-3.1.4.RELEASE.jar | 838801 |
| spring-context-support-3.1.4.RELEASE.jar | 107164 |
| spring-core-3.1.4.RELEASE.jar | 451269 |
| spring-expression-3.1.4.RELEASE.jar | 179323 |
| spring-jdbc-3.1.4.RELEASE.jar | 405635 |
| spring-ldap-core-1.3.1.RELEASE.jar | 231729 |
| spring-security-config-3.1.0.RELEASE.jar | 202754 |
| spring-security-core-3.1.0.RELEASE.jar | 348567 |
| spring-security-crypto-3.1.0.RELEASE.jar | 41068 |
| spring-security-ldap-3.1.0.RELEASE.jar | 93631 |
| spring-security-web-3.1.0.RELEASE.jar | 255577 |
| spring-tx-3.1.4.RELEASE.jar | 245483 |
| spring-web-3.1.4.RELEASE.jar | 554802 |
| spring-webmvc-3.1.4.RELEASE.jar | 579461 |
| stax-1.2.0.jar | 179346 |
| stax-api-1.0.1.jar | 26514 |
| stream-2.7.0.jar | 174351 |
| tachyon-client-0.7.1.jar | 1972071 |
| tachyon-underfs-hdfs-0.7.1.jar | 11390 |
| tachyon-underfs-local-0.7.1.jar | 7318 |
| uncommons-maths-1.2.2a.jar | 49019 |
| unused-1.0.0.jar | 2777 |
| wicket-1.4.12.jar | 1903610 |
| wicket-extensions-1.4.12.jar | 1179862 |
| wicket-ioc-1.4.12.jar | 23286 |
| wicket-spring-1.4.12.jar | 29377 |
| xml-apis-1.4.01.jar | 220536 |
| xml-apis-ext-1.3.04.jar | 85686 |
| xml-commons-resolver-1.2.jar | 84091 |
| xmlpull-1.1.3.1.jar | 7188 |
| xpp3-1.1.3.4.O.jar | 119888 |
| xpp3_min-1.1.4c.jar | 24956 |
| xsd-2.6.0.jar | 992820 |
| xstream-1.4.7.jar | 531571 |
| xz-1.0.jar | 94672 |
| zookeeper-3.4.5.jar | 779974 |