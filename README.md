## EEA Semantic Data Service docker image

### 1. Deployment
__1.1 Getting the latest release up__
```
shell> git clone https://github.com/eea/eea.docker.sds.git 
       git clone https://github.com/eea/eea.docker.virtuoso.git
       cd eea.docker.virtuoso
```
In order to configure the Virtuoso, one needs to rename the example file in __virtuoso.ini__ and set the parameters to needed settings.
Also for using docker-compose you need to copy the file from eea.docker.sds in eea.docker.virtuoso, rename the example and set the path for shared folders

__1.2 Setting up shared folders, required for uploading files and harvesting rdfs__

To actually use a shared folder between Virtuoso and SDS / CR app, you have to set the folders from volumes zone in docker-compose file :
```
volumes:
  - /folder_host/local/cr3/files:/shared_folder/local/cr3/files:z
  - /folder_host/backups/sql:/shared_folder/backups/sql:z
  - /folder_host/tmp:/shared_folder/tmp:z
```
After, if needed, specify updated path folders in __virtuoso.ini__ file on DirsAllowed parameters.

If you need to modify the default path of temporary database, you can add a busybox data container with the needed path inside container and change the Databasefile parameter from Tempdatabase section from __virtuoso.ini__ file.
Or, if you want only a folder on host, add a volume in virtuoso container, mapped to specified folder, inside the docker-compose file.

__1.3 Starting Virtuoso container for the first time__
```
shell> docker-compose up -d --no-recreate
```

### 2. Migrating existing data
__2.1__ Remove the default data files
```
shell> docker exec eeadockervirtuoso_virtuoso_1 find /virtuoso_db/ -type f ! -name '*.ini' -delete
```
__2.2__ Copy the existing virtuoso.db from the mounted volume with data
```
docker run --rm \
  --volumes-from eeadockervirtuoso_datav_1 \
  --volume <mounted-data-folder>/virtuoso.db:/restore/virtuoso.db:ro \
busybox \
  sh -c "cp -r /restore/virtuoso.db /virtuoso_db && \
  chown -R 500:500 /virtuoso_db"
```
__2.3.__ Restart the container
```
shell> docker-compose stop
       docker-compose up -d --no-recreate
```
Next, we need to build the SDS application.

### 3. Install Java, Apache Tomcat and Maven
SDS runs on the Java platform, and has been tested to run on Tomcat Java Servlet Container. SDS source code is built with Maven.
Please download all of these software and install them according to the instructions found at their websites.
The necessary versions are as follows:
* Java 1.6 or 1.7 ( higher versions are not recommended due to incompatibility ) 
* Maven 3.0.5 or higher
* Tomcat 6.0 or higher

### 4. Download, configure and install SDS

Clone SDS source code into denoted directory SDS_SOURCE_HOME in the below instructions.
```
shell> git clone https://github.com/eea/eionet.contreg.git
```
Before you can build SDS source code, you need to set your environment specific properties. For that, make a copy of unittest.properties in SDS_SOURCE_HOME, and rename it to local.properties. Go through the resulting file and change properties that are specific to your environment or wishes. Each property's exact meaning and effect is commented in the file.

Now you are ready to build your SDS code. It is built with Maven. The following command assumes that Maven's executable (mvn) is on the command path, and that it is run while being in SDS_SOURCE_HOME directory:
```
shell> mvn -Dmaven.test.skip=true  clean install
```

### 5. Deploy SDS web application and run Tomcat

If the build went well, you shall have cr.war file in SDS_SOURCE_HOME/target directory. Now you have to do is to simply copy that file into Tomcat's webapps directory. Optionally, you can also deploy the WAR file via Tomcat's web console, but be sure to have made the following Tomcat configuration trick, before running Tomcat.

Before you run Tomcat, you need to change the way Tomcat handles URI encoding. By default, it uses ISO-8859-1 for that. But SDS needs UTF-8. Therefore make sure that the tag in Tomcat's server.xml has the following attributes:

```
URIEncoding="UTF-8"
useBodyEncodingForURI="true"
```

Once Tomcat is running, open SDS in the browser. It's application context path is cr/, unless you renamed cr.war to something else or you chose to deploy SDS into a virtual host.
