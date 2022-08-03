# Validate Hadoop Release Artifacts
l
This project helps validate hadoop release candidates

It has an ant `build.xml` file to help with preparing the release,
validating gpg signatures, creating release messages and other things.

# ant builds

see below



# maven builds

To build and test with the client API:

```bash
mvn clean test 
```

Compilation verifies the API is present; the
test verifies that some shaded artifacts are present.

If the hadoop artifacts are in staging/snapshot repositories,
use the `staging` profile

```bash
mvn clean test -Pstaging
```

To force an update

```bash
mvn clean test -Pstaging -U
```

To purge all artifacts of the chosen hadoop version from your local maven repository.

```bash
ant purge
```

# workflow for preparing an RC

Build the RC using the docker process on whichever host is set to do it

### set up build.properties

```properties
scp.hostname=stevel-ubuntu
scp.user=stevel
scp.hadoop.dir=hadoop
staging.dir=/Users/stevel/hadoop/release/staging
spark.dir=/Users/stevel/Projects/sparkwork/spark
cloud-examples.dir=/Users/stevel/Projects/sparkwork/cloud-integration/cloud-examples
cloud.test.configuration.file=/Users/stevel/Projects/config/cloud-test-configs/s3a.xml
bigdata-interop.dir=/Users/stevel/Projects/gcs/bigdata-interop
hboss.dir=/Users/stevel/Projects/hbasework/hbase-filesystem
cloudstore.dir=/Users/stevel/Projects/cloudstore
hadoop.version=3.3.4
rc=0
```

### Clean up first


```bash
ant clean
```

And then purge all artifacts of that release from maven.
This is critical when validating downstream project builds.

```bash
ant purge-from-maven
```

### Download RC to `target/incoming`

This will take a while! look in target/incoming for progress

```bash
ant scp-artifacts
```


### Move to the release dir


```bash
ant move-scp-artifacts release.dir.check
```

### verify gpg signing


```bash
ant gpg.keys gpg.verify
```

### copy to a staging location in the hadoop SVN repository.

When committed to svn it will be uploaded and accessible via an
https://svn.apache.org URL.

```bash
ant stage
```
When the RC is released, an `svn move` operation can promote it
directly.

### In the staging svn repo, update, add and commit the work

This is not part of the tool. Can take a while...exit any VPN for extra speed.

```bash
svn update
svn add <RC directory name>
svn commit 
```

### tag the rc and push to github

This isn't automated as it needs to be done in the source tree.

```bash
ant print-tag-command
```

### Prepare for the email

1. Go to https://repository.apache.org/#stagingRepositories
2. Find the hadoop repo for the RC
3. "close" it and wait for that to go through


### Generate the RC vote email

Review/update template message in `src/email.txt`.
All ant properties referenced will be expanded if set.

```bash
ant vote-message
```

The message is printed and saved to the file `target/email.txt`

*do not send it until you have validated the URLs resolve*

Now wait for the votes to come in. This is a good time to
repeat all the testing of downstream projects, this time
validating the staged artifacts, rather than any build
locally.

# How to download and build someone else's release candidate

In build properties, declare `hadoop.version`, `rc` and `http.source`

```properties
hadoop.version=3.3.4
rc=1
http.source=https://dist.apache.org/repos/dist/dev/hadoop/hadoop-3.3.4-RC1/
```

targets of relevance

| target             | action                     |
|--------------------|----------------------------|
| release.fetch.http | fetch artifacts            |
| release.dir.check  | verify release dir exists  |
| release.src.untar  | untar retrieved artifacts  |
| release.src.build  | build the source           |
| release.src.test   | build and test the source  |
| gpg.keys           | import the hadoop KEYS     |
| gpg.verify         | verify the D/L'd artifacts |
|                    |                            |


set `release.native.binaries` to false to skip native binary checks on platforms without them

### Download the RC files from the http server

```bash
ant release.fetch.http
```

### untar and build.

This puts the built artifacts into the local maven repo so
do not do this while building/testing downstream projects
*and call `ant purge-from-maven` after*

```bash
ant release.src.untar release.src.build
```


# Building and testing projects from the staged maven artifacts

A lot of the targets build maven projects from the staged maven artifacts.

For this to work
1. check out the relevant projects somewhere
2. set their location in the build.properties file
3. make sure that the branch checked out is the one you want to build.
   This matters for anyone who works on those other projects
   on their own branches.
4. Some projects need java11.

First, purge your maven repo

    
```bash
ant purge-from-maven
```

## Cloudstore

No tests, sorry.

```
ant cloudstore.build
```

## Google GCS

This is java11 only.
 
Ideally, you should run the tests, or even better, run them before the RC is up for review.


Building the libraries.
Do this only if you aren't running the tests.

```
ant gcs.build
```

Testing the libraries
```
ant gcs.build
```



## Apache Spark

Validates hadoop client artifacts; the cloud tests cover hadoop cloud storage clients.


```bash
ant spark.build
```

Then followup cloud examples if you are set up
```bash
ant cloud-examples.build
ant cloud-examples.test
```

## HBase filesystem


```bash
ant hboss.build
```

