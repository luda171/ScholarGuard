# ScholarGuard
Institutional Framework for Tracking and Archiving Research Artifacts
## Orchestrator

The Orchestrator manages the scheduling of the user events for the tracker, crawler, and the archiver. The main 
components to make the Orchestrator work are:

* The Orchestrator: This component is written in Python. Includes:
  * The UI of myresearch.institute.
  * Maintains the User DB that contains information about users in various portals. Uses SQLite. 
  * Maintains the Events DB that stores the events from Tracker, Crawler, and Archiver. Uses ElasticSearch.
  * Triggers the Tracker according to a pre-set schedule for the various users. Uses Celery for scheduling.
  * Includes the LDN Inbox to send and receive messages from the various components. Written in Python.
* ElasticSearch: The database that stores all the events and makes them searcheable. 
* SQLite: This database stores all the user information along with their portal credentials.
* Celery/Redis: This manages the scheduling tasks of the orchestrator. The Redis database maintaines the queue. 

All these components are dockerized and can be started with one `docker-compose` command.

### Requirements
  * [Docker](https://docs.docker.com/)
  * [Docker-compose](https://docs.docker.com/compose/)
  * Python 3, python3 virtualenv
  
### Installation
Please refer to the corresponding docs for the items listed in requirements for installation instructions. 
**NOTE**: The `docker` and `docker-compose` commands may need the `sudo` prefix.

#### To deploy from scratch:
```
assuming we clone ScholarGuard to /data/web/
$> cd /data/web/ScholarGuard

$> cd orchestrator

# build the docker containters
$> docker-compose build

# create secrets so that the orchestrator can encrypt the user's credentials
$> mkdir secrets
$> ./create_secrets

# create data directories and make them writable for the ES and sqlite processes 
# running inside the container
$> mkdir ./data/es
$> mkdir ./data/es2
$> mkdir ./data/sql
$> chown -R 1000:1000 data/es*
$> chown -R 1000:1001 data/sql
$> chmod -R 777 data/*

# If elasticsearch complains that the max virtual memory is too low:
$> sudo sysctl -w vm.max_map_count=262144


# start the orchestrator
# the -d option starts the services in the background as a daemon
$> docker-compose up -d
```

To create the virtual environment:
```
$> cd /data
$> mkdir venv
$> cd venv
$> virtualenv -p /usr/bin/python3 orchestrator
$> cd /data/web/orchestrator/web
$> python setup.py install
$> pip install -e .
```

**Config**

In the file `conf/config_standalone.yaml`, please edit the field `db:sqlalchemy_database_uri` to point to the appropriate file 
path in the new server.

In the file `conf/config.yaml`, please edit the fields `pod:server_name, 
pod:org_full_name, pod:preferred_url_scheme` to reflect the new changes. 


To add new users to the pipeline, run the command below from inside the virtualenv. Please make sure the 
elasticsearch instances are running. 
```
$> ./bin/add_or_update_users_in_db
```



**IMPORTANT**

* Please edit the variable `users_csv_file` in the script above to point to the correct CSV file conatining user 
  details. 
* The script above expects the CSV file columns to be in the order below:
`id,user_types,full_name,profile_image,orcid_username,orcid_user_id,twitter_username,twitter_user_id,github_username,github_user_id,slideshare_username, slideshare_user_id,hypothesis_username,hypothesis_user_id,stackoverflow_username,stackoverflow_user_id,wikipedia_username,wikipedia_user_id,figshare_username,figshare_user_id,medium_username,medium_user_id,blogger_username,blogger_user_id,publons_username,publons_user_id`
* This same script will **OVERWRITE** any user portals for existing users in the database by matching the orcids. 


#### To deploy bug fixes or modifications:

Commit the changes made to the code in the above repository and pull the changes to the deployment directory in the server.
In the server:
```
$> cd /data/web/ScholarGuard/orchestrator
$> git pull
$> docker-compose build
$> docker-compose stop
$> docker-compose up -d
```

Starting the orchestrator via Docker will start all the necessary databases, and will 
bring up all the necessary components. 

### Endpoints and Directories in the Server
Assuming that The orchestrator is in the server `myresearch.institute`.

The Orchestrator is deployed in the directory `/data/web/ScholarGuard/orchestrator`. 

The main end point for the orchestrator is `https://myresearch.institute`. 
The orchestrator maintains different LDN inboxes for the 3 different components. 
* The Tracker inbox is at `https://myresearch.institute/orchestrator/tracker/inbox/`.
* The Capture inbox is at `https://myresearch.institute/orchestrator/capture/inbox/`.
* The Archiver inbox is at `https://myresearch.institute/orchestrator/archiver/inbox/`.


### Start/Stop/Restart

* Start: `docker-compose up -d`
* Stop: `docker-compose down`
* Restart: `docker-compose restart`

When updating the code base, the containers should be stopped and started individually instead of using the restart command. 

### Troubleshooting
* `docker ps` will list all the running docker containers. There are 3 containers that should be running for the orchestrator 
to be working correctly: (1) `pod-app` runs the Orchestrator Python app, (2) `pod-es` and (3) `pod-es2` both run the elasticsearch 
database. 
* The SQLite user database is on disk in the folder `./data/sql/users_portals.db`. This file should be writable by the user starting 
the containers. 
* The logs of all the containers can be seen by `docker-compose logs`. To continuously output the logs, 
`docker-compose logs --tail=200 -f".
* To resend any missed events to the archiver or capture process:
```
$> ./bin/resend_missed_events_to_archiver
$> ./bin/resend_missed_events_to_capture
```
This command must be run from within the virtualenv:
```
$> source /data/venv/orchestrator/bin/activate
$> cd /data/web/orchestrator/web/
$> python setup.py install
$> cd ../
$> ./bin/resend_missed_events_to_capture
```
This command depends on the settings in the file `conf/config_standalone.yaml`. Please make sure the details there are accurate. 

------------
## Tracker

The tracker polls various portals for updates from various users. The tracker consists of:

* Portal Trackers: One tracker per portal that checks for updates for a user and sends an event message to the orchestrator.
* LDN Inbox: The inbox that listens to messages to track a user in a portal from the orchestrator. The tracker activates the 
appropriate portal tracker once this message is received. 
* SQLite: The LDN inbox stores the incoming tracking requests in this db. 

All these components are dockerized and can be started with one `docker-compose` command.

### Requirements
* [Docker](https://docs.docker.com/)
* [Docker-compose](https://docs.docker.com/compose/)
* Python 3, python3 virtualenv

### Installation
Please refer to the corresponding docs for the items listed in requirements for installation instructions. 

### Deployment
The repository for the tracker is at:
./artifact_tracker/

**NOTE**: The `docker` and `docker-compose` commands may need the `sudo` prefix.

To deploy from scratch:
```
$> cd /data/web/ScholarGuard/

$> cd artifact_tracker
$> docker-compose build
# the -d option starts the services in the background as a daemon
$> docker-compose up -d
```
To deploy bug fixes or modifications:

Commit the local changes made to the code in the above repository, and pull the changes to the deployment directory in the server.
In the server:
```
$> cd /data/web/ScholarGuard/artifact_tracker
$> git pull
$> docker-compose build
$> docker-compose stop
$> docker-compose up -d
```

Starting the tracker via Docker will start all the necessary components.

### Endpoints and Directories in the Server
Assuming the tracker is in the server `myresearch.institute`.

The tracker is deployed in the directory `/data/web/ScholarGuard/artifact_tracker`. 

The only HTTP end point for the tracker is the LDN inbox at `https://myresearch.institute/tracker/inbox/`.


### Start/Stop/Restart

* Start: `docker-compose up -d`
* Stop: `docker-compose down`
* Restart: `docker-compose restart`

When updating the code base, the containers should be stopped and started individually instead of using the restart command. 

### Troubleshooting
* `docker ps` will list the running docker containers. `tracker-app` is the name of the tracker container that should be running.
* The SQLite user database is on disk in the folder `./data/sql/users_portals.db`. This file should be writable by the user starting 
the containers. 
* The logs of all the containers can be seen by `docker-compose logs`. To continuously output the logs, 
`docker-compose logs --tail=200 -f".
* To test if the inbox is running, a simple curl command should provide a similar response to the one below with the 
`Link`, `Allow` and `Accept-Post` headers as per the LDN spec. 
```
$> curl -I https://myresearch.institute/tracker/inbox/

HTTP/1.1 200 OK
Server: nginx/1.12.1
Date: Mon, 24 Jun 2019 21:56:53 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 0
Connection: keep-alive
Allow: GET, HEAD, OPTIONS, POST
Link: <https://myresearch.institute/tracker/inbox/>; rel="https://www.w3.org/ns/ldp#inbox", <http://www.w3.org/ns/ldp#Resource>; rel="type",<http://www.w3.org/ns/ldp#RDFSource>; rel="type",<http://www.w3.org/ns/ldp#Container>; rel="type",<http://www.w3.org/ns/ldp#BasicContainer>; rel="type"
Accept-Post: application/ld+json
```

----------------

## Capture 
The Capture system consists of two main modules: Capture Inbox and Capture Crawler. The source code for both modules is available in the stormarchiver repository. The Capture Crawler is built using the Stormcrawler framework, which operates on an Apache Storm topology. A topology is a directed graph of data stream transformations, where each node functions as either a spout or a bolt.

I developed a custom SQLSpoutInput spout to associate each URL with a specific tracker event ID. Additionally, a custom StatusUpdaterBolt is used to conclude the topology by inserting extra metadata, such as sub-trace information, into the MySQL database. The trace execution is handled by the Navigation Filter plugin.

The system also features a custom Selenium protocol plugin that extends the Stormcrawler AbstractHttpProtocol class. It spawns a Warcproxy process for each URL, creating a new Remote WebDriver connection to a Selenium hub. The Selenium hub is deployed in a Docker container, and the WebDriver instances communicate with the hub using the JSON Wire Protocol over HTTP.


### Requirements  Installation  Deployment
see capture_inbox/
and
stormarchiver/

### Endpoints and Directories in the Server

The Capture Inbox (CI) is a server and deployed in the directory `/data/web/capture` (CI directory).
CI is configured with 
```
warcbaseurl =https://myresearch.institute/capture/warc/
posturl = https://scholarlyorphans.org/inbox/
trackerposturl=https://myresearch.institute/tracker/inbox/
capturebaseurl=https://myresearch.institute/capture/
```
and logs to the ` /data/web/capture/log/wrapper.log ` .
The Capture Crawler resides in the directory  `/data/web/stormarchiver` (CC directory).
It runs continuously and currently it logs to the  000storm.out file.
Config Information  crawler-conf.yaml . More information can be found under stormarchiver bitbucket repository.

### Start/Stop/Restart
to start Capture Inbox: go to the CI directory.
```
./bin/trxapp stop
./bin/trxapp start
```
to start Capture Crawler : go to the CC directory and do:
```
nohup storm jar target/stormcrawler-0.1.jar  -Djava.io.tmpdir=/data/tmp  org.apache.storm.flux.Flux    crawler.flux -s 10000000000 > 000storm.out &
```
to kill crawler process -- search pid of the process
```
ps aux|grep storm
ludab     55863  6.9  2.8 42674952 3630212 ?    Sl   Jun04 1428:27 /usr/lib/jvm/java/bin/java -client -Ddaemon.name= -Dstorm.options= -Dstorm.home=/usr/local/apache-storm-1.2.2 -Dstorm.log.dir=/usr/local/apache-storm-1.2.2/logs -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib -Dstorm.conf.file= -cp /usr/local/apache-storm-1.2.2/*:/usr/local/apache-storm-1.2.2/lib/*:/usr/local/apache-storm-1.2.2/extlib/*:target/stormcrawler-0.1.jar:/usr/local/apache-storm-1.2.2/conf:/usr/local/apachestorm-1.2.2/bin -Dstorm.jar=target/stormcrawler-0.1.jar -Dstorm.dependency.jars= -Dstorm.dependency.artifacts={} -Djava.io.tmpdir=/data/tmp org.apache.storm.flux.Flux crawler.flux -s 10000000000
ludab    102487  0.0  0.0 110516  2116 pts/6    S+   21:08   0:00 grep --color=auto storm
kill -9  55863
```
### Troubleshooting
Check the mysql db (connection info in crawler-conf.yaml). If all urls got processed, will be  no "DISCOVERED" status. 
```
select status,count(*) from urls group by status;
```
------------------

## Archiver
There are four components that are interdependent to varying degrees, and so the order in which the components should be started is important. The pydldn inbox should be started first. This is event messages intended for the archiver are received. This are written to a mysql table and subsequently processed by the archiver. Next, start the pywb replay application. This is necessary both to respond to memento requests and for archiver to access the cdxj API so that it can discover the memento for a newly retrieved WARC for a capture event. Next, start the wayback autoindex process. This ensures that newly added WARC files will be indexed. Finally, start the archiver process. Archiver processes messages in the archiver inbox, retrieves WARC(s) specified in the capture event an stores them in the memento/collections/archive directory, queries the cdxj index for memento(s) for the newly added WARC, and finally publishes an archive message so that the archive process for a given message can be marked as complete. It wakes up every five minutes to check the archiver inbox. 
### Requirements
Python libraries required by archiver inbox and the archiver process are associated with the researcher_pod virtualenv in /data/venv/researcher_pod

Pywb relies on libraries installed inthe archiver virtualenv in /data/venv/archiver

### Installation
### Deployment

### Endpoints and Directories in the Server
* Archiver inbox is located in /data/web/archiver/inbox
* The archiver processor is located in /data/web/archiver/processor
* Pywb is located in /data/web/archiver/replay

### Start/Stop/Restart
#### Step 1 - start archiver inbox
* depends on libraries found in the researcher_pod virtualenv
* its home directory is: /data/web/archiver/inbox
* to start: nohup uwsgi -H /data/venv/archiver_inbox uwsgi.ini &

#### Step 2 - start pywb
* set archiver web environment:
* source /data/venv/archiver/bin/activate
* depends on libraries found in the archiver virtualenv
* its home directory is: /data/web/archiver/replay
* to start: nohup uwsgi uwsgi.ini &

#### Step 3 - start wayback reindexing process
* depends on libraries found in the archiver virtualenv
* its home directory is : /data/web/archiver/replay
* to start: /data/venv/archiver/bin/python /data/venv/archiver/bin/wayback -a --auto-interval 30 & 

#### Step 4 - start the archiver processor
* Start the archiver last. Note that the archiver is not a Web process, it is just a python application.
* depends on libraries found in the researcher_pod virtualenv
* its home directory is: /data/web/archiver/processor
* to start: nohup python eventArchiver.py & 

### Troubleshooting

To troubleshoot the archiver inbox, check /data/web/archiver/inbox/nohup.out and also check running processes for multipe processes as described above. You will also want to check the Mysql table associated with the archiver inbox, whcih is called "messages". If there are items in the inbox that have not yet been processed, this query will show them:

select * from messages where date_processed is null;

If there are outstanding items that have not been processed by the archiver, and they do not appear in the messages table, then the archiver inbox will have to be restarted and those messages will have to be posted to the archiver inbox again. The archiver processor and Pywb are not affected by issues with the inbox. Once items appear in the inbox, they will be processed by the archiver and when the warcs are retrieved, they will be indexed by the autoindex process. 

To troubleshoot Pywb, first check the collection directory for new warcs. This directory is located in /data/web/archiver/replay/collections/memento/archive. If the expected warcs have been written to this directory but there still appears to be a problem, check the timestamp for /data/web/archiver/replay/collections/memento/indexes/autoindex.cdxj. If this file has not be updated since new warcs were written, then the autoindex process will need to be restarted. Care should be taken not to overwrite this index as this will force a complete reindex of all warcs, which could take many hours. 

To troubleshoot the archiver processor, check the log file /data/var/logs/archiver.log. The archiver processor wakes up every five minutes to query the messages table for new messages. If new items arrive after this query occurs, then these items will not be processed until the current batch is completed. Only then will the processor requery the messages table. 

## MementoEmbed

[MementoEmbed](https://github.com/oduwsdl/MementoEmbed) is an external application that creates cards for mementos. It runs from a Docker container. To start MementoEmbed, do the following:

`
sudo docker run -d -p 5550:5550 oduwsdl/mementoembed
`
