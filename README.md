# Validate Hadoop Release Artifacts

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
fs-api-shim.dir=/Users/stevel/Projects/Formats/fs-api-shim/
hadoop.version=3.3.5
git.commit.id=3262495904d
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

### Copy to the release dir

Copies the files from `downloads/incoming/artifacts` to `downloads/hadoop-$version-$rc`'

```bash
ant copy-scp-artifacts release.dir.check
```

The `release.dir.check` target just lists the directory.

### Arm64 binaries

If arm64 binaries are being created then they must be
built on an arm docker image.
Do not do this at the same time as building the x86 binaries
because both builds will generate staging repositories on
nexus. Instead: run the arm one first and drop its artifacts
on nexus before doing the x86 one. That will ensure that
it is the JAR files created on the x86 build are the ones
publised on maven.

The arm process is one of
1. Create the full set of artifacts on an arm machine (macbook, cloud vm, ...)
2. Drop any mvn repository from nexus
3. Use the ant build to copy and rename the .tar.gz with the native binaries only
4. Create a new .asc file. This is needed is without the `--asfrelease` option no signing takes place.
5. Generate new sha512 checksum file containing the new name.
6. Move these files into the `downloads/release/$RC` dir

To perform stages 3-6:
```bash
ant arm.copy.artifacts arm.release
```


### verify gpg signing

```bash
ant gpg.keys gpg.verify
```

### copy to a staging location in the hadoop SVN repository.

When committed to subversion it will be uploaded and accessible via a
https://svn.apache.org URL.


*do this after preparing the arm64 binaries*

Final review of the release files
```bash
ant release.dir.check
```

Now stage
```bash
ant stage
```

This makes it visible to others via the apache svn site, but it
is not mirrored yet.

When the RC is released, an `svn move` operation can promote it
directly.

### In the staging svn repo, update, add and commit the work

Can take a while...exit any VPN for extra speed.

#### Manual
```bash
cd $staging-dir
svn update
svn add <RC directory name>
svn commit
```

#### Ant

```bash
ant stage-to-svn
```



### tag the rc and push to github

This isn't automated as it needs to be done in the source tree.

```bash
ant print-tag-command
```

### Prepare the maven repository

1. Go to https://repository.apache.org/#stagingRepositories
2. Find the hadoop repo for the RC
3. "close" it and wait for that to go through

### Generate the RC vote email

Review/update template message in `src/text/vote.txt`.
All ant properties referenced will be expanded if set.

```bash
ant vote-message
```

The message is printed and saved to the file `target/vote.txt`

*do not send it until you have validated the URLs resolve*

Now wait for the votes to come in. This is a good time to
repeat all the testing of downstream projects, this time
validating the staged artifacts, rather than any build
locally.

# How to download and build someone else's release candidate

In build properties, declare `hadoop.version`, `rc` and `http.source`

```properties
hadoop.version=3.3.5
rc=1
http.source=https://dist.apache.org/repos/dist/dev/hadoop/hadoop-${hadoop.version}-RC${rc}/
```

targets of relevance

| target               | action                     |
|----------------------|----------------------------|
| `release.fetch.http` | fetch artifacts            |
| `release.dir.check`  | verify release dir exists  |
| `release.src.untar`  | untar retrieved artifacts  |
| `release.src.build`  | build the source           |
| `release.src.test`   | build and test the source  |
| `gpg.keys`           | import the hadoop KEYS     |
| `gpg.verify `        | verify the D/L'd artifacts |
|                      |                            |

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

1. Check out the relevant projects somewhere
2. Set their location in the `build.properties` file
3. Make sure that the branch checked out is the one you want to build.
   This matters for anyone who works on those other projects
   on their own branches.
4. Some projects need java11.

First, purge your maven repo

```bash
ant purge-from-maven
```

## client validator maven


```bash
ant mvn-test
```

## Cloudstore

[cloudstore](https://github.com/steveloughran/cloudstore).

No tests, sorry.

```bash
ant cloudstore.build
```

## Google GCS


[Big Data Interop](https://github.com/GoogleCloudPlatform/bigdata-interop).

This is java 11+ only.

Ideally, you should run the tests, or even better, run them before the RC is up for review.

Building the libraries.
Do this only if you aren't running the tests.

```bash
ant gcs.build
```

## Apache Spark

Validates hadoop client artifacts; the cloud tests cover hadoop cloud storage clients.

```bash
ant spark.build
```

### Spark cloud integration tests

Then followup cloud integration tests if you are set up to build.
Spark itself does not include any integration tests of the object store connectors.
This independent module tests the s3a, gcs and abfs connectors,
and associated committers, through the spark RDD and SQL APIs.

[cloud integration](https://github.com/hortonworks-spark/cloud-integration)
```bash
ant cloud-examples.build
ant cloud-examples.test
```

## HBase filesystem

[hbase-filesystem](https://github.com/apache/hbase-filesystem.git)

Adds zookeeper-based locking on those filesystem API calls for which
atomic access is required.

Integration tests will go through S3A connector.

```bash
ant hboss.build
```

## building the Hadoop site

Set `hadoop.site.dir` to be the path of the
local clone of the ASF site repository
https://gitbox.apache.org/repos/asf/hadoop-site.git

```properties
hadoop.site.dir=/Users/stevel/hadoop/release/hadoop-site
```

Prepare the site with the following targets

```bash
ant release.site.announcement
ant release.site.docs
```

Review the annoucement.

### Manually link the current/stable symlinks to the new release

In the hadoop site dir

```bash

# review current status
ls -l

# symlink current
rm current3
ln -s r.3.3.5 current3

# symlink stable
rm stable3
ln -s r3.3.5 stable
ln -s r3.3.5 stable3

# review new status
ls -l
```

### Git status prompt issues in fish

There are a lot of files, and if your shell has a prompt which shoes the git repo state, scanning can take a long time.
Disable it, such as for fish:

```fish
set -e __fish_git_prompt_showdirtystate
```

Finally, *commit*

## Adding a global maven staging profile `asf-staging`

Many projects have a profile to use a staging repository, especially the ASF one.

Not all do -these builds are likely to fail.
Here is a profile, `asf-staging` which can be used to enable this.
The paths to the repository can be changed too, if desired.

Some of the maven builds invoked rely on this profile (e.g. avro).
For some unknown reason the parquet build doesn't seem to cope.

```xml
 <profile>
  <id>asf-staging</id>
  <properties>
    <!-- override point for ASF staging/snapshot repos -->
    <asf.staging>https://repository.apache.org/content/groups/staging/</asf.staging>
    <asf.snapshots>https://repository.apache.org/content/repositories/snapshots/</asf.snapshots>
  </properties>

  <pluginRepositories>
    <pluginRepository>
      <id>ASF Staging</id>
      <url>${asf.staging}</url>
    </pluginRepository>
    <pluginRepository>
      <id>ASF Snapshots</id>
      <url>${asf.snapshots}</url>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
      <releases>
        <enabled>false</enabled>
      </releases>
    </pluginRepository>

  </pluginRepositories>
  <repositories>
    <repository>
      <id>ASF Staging</id>
      <url>${asf.staging}</url>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
      <releases>
        <enabled>true</enabled>
      </releases>
    </repository>
    <repository>
      <id>ASF Snapshots</id>
      <url>${asf.snapshots}</url>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
      <releases>
        <enabled>true</enabled>
      </releases>
    </repository>
  </repositories>
</profile>

```
