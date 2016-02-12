## EEA Semantic Data Service docker image

### 1. Deployment
__1.1 Getting the latest release up__

```
shell> git clone https://github.com/eea/eea.docker.sds.git
       cd eea.docker.sds
```
The Virtuoso is already configured. In order to reconfigure the Virtuoso, one needs to use file  __virtuoso.ini__ and set the parameters to needed settings.

__1.2 Setting up shared folders, required for uploading files and harvesting rdfs__

The docker-compose file is configured to use established shared folders. To actually use a different shared folder between Virtuoso and SDS app, you have to set them from volumes zone:
```
    - /var/local/cr3/files:/var/local/cr3/files:z
    - /var/backups/sql:/var/backups/sql:z
    - /var/tmp:/var/tmp:z
```
After, if needed, specify updated path folders in __virtuoso.ini__ file on DirsAllowed parameters.

If you need to modify the default path of temporary database, you can add a busybox data container with the needed path inside container and change the Databasefile parameter from Tempdatabase section in __virtuoso.ini__ file.
Or, if you want only a folder on host, add a volume in virtuoso container, mapped to specified folder, inside the docker-compose file.

__1.3 Development setup__

For development purposes you need to expose port 1112, in order to run integration tests in Maven.

__1.4 Starting Virtuoso container for the first time__
```
shell> docker-compose up
```

### 2. Migrating existing data
__2.1 Remove the default data files__

In other terminal, you have to delete all the files related to data:
```
shell> docker exec eeadockersds_virtuoso_1 find /virtuoso_db/ -type f ! -name '*.ini' -delete
```
__2.2 Copy the existing virtuoso.db from the mounted volume with data__
```
docker run --rm \
  --volumes-from eeadockersds_virtuoso_data_1 \
  --volume <mounted-data-folder>/virtuoso.db:/restore/virtuoso.db:ro \
busybox \
  sh -c "cp -r /restore/virtuoso.db /virtuoso_db && \
  chown -R 500:500 /virtuoso_db"
```
__2.3 Restart the container__
```
shell> docker-compose stop
       docker-compose up -d --no-recreate
```
Now you have docker container with Virtuoso, connected to datacontainer with latest database.
Next, we need to build the SDS application.

### 3. Install Java, Apache Tomcat and Maven
SDS runs on the Java platform, and has been tested to run on Tomcat Java Servlet Container. SDS source code is built with Maven.
Please download all of these software and install them according to the instructions found at their websites.
The necessary versions are as follows:
* Java 1.6 or 1.7 ( higher versions are not recommended due to incompatibility ) 
* Maven 3.0.5 or higher
* Tomcat 6.0 or higher

### 4. Download, configure and install SDS

Clone SDS source code into /var/local/deploy directory with next instruction.
```
shell> git clone https://github.com/eea/eionet.contreg.git
       cd eionet.contreg/
```
Before you can build SDS source code, you need to set your environment specific properties. For that, make a copy of unittest.properties in ```eionet.contreg/```, and rename it to local.properties. Go through the resulting file and change properties that are specific to your environment or wishes. Each property's exact meaning and effect is commented in the file.

Now you are ready to build your SDS code with Maven. The following command assumes that Maven's executable (mvn) is on the command path, and that it is run while being in ```eionet.contreg/``` directory:
```
shell> mvn -Dmaven.test.skip=true  clean install
```

### 5. Deploy SDS web application and run Tomcat

If the build went well, the cr.war file will be available in ```eionet.contreg/target/``` directory. You have to do is to simply copy that file into Tomcat's ```webapps/``` directory. Rename it with ROOT.war if there isn't any other application in ```webapps/``` folder.

Before you run Tomcat, you need to change the way Tomcat handles URI encoding. By default, it uses ISO-8859-1, but SDS needs UTF-8. Therefore make sure that the tag ```<Connector/>``` from server.xml has the following attributes:

```
URIEncoding="UTF-8"
useBodyEncodingForURI="true"
```

Once Tomcat is running, open SDS in the browser. It's application context path is cr/, or /, unless you renamed cr.war.
