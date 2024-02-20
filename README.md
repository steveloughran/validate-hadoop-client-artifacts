# Validate Hadoop Release Artifacts

This project helps validate hadoop release candidates

It has an ant `build.xml` file to help with preparing the release,
validating gpg signatures, creating release messages and other things.

# workflow for preparing an RC

Build the RC using the docker process on whichever host is set to do it.

### set up `build.properties`

```properties
hadoop.version=3.3.5
# RC for emails, staging dir names
rc=0

# id of commit built; used for email
git.commit.id=3262495904d

# info for copying down the RC from the build host
scp.hostname=stevel-ubuntu
scp.user=stevel
scp.hadoop.dir=hadoop

# SVN managed staging dir
staging.dir=/Users/stevel/hadoop/release/staging

# where various modules live for build and test
spark.dir=/Users/stevel/Projects/sparkwork/spark
cloud-examples.dir=/Users/stevel/Projects/sparkwork/cloud-integration/cloud-examples
cloud.test.configuration.file=/Users/stevel/Projects/config/cloud-test-configs/s3a.xml
bigdata-interop.dir=/Users/stevel/Projects/gcs/bigdata-interop
hboss.dir=/Users/stevel/Projects/hbasework/hbase-filesystem
cloudstore.dir=/Users/stevel/Projects/cloudstore
fs-api-shim.dir=/Users/stevel/Projects/Formats/fs-api-shim/

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
Do not use the `--asfrelease` option as this stages the JARs.
Instead use the explicit `--deploy --native --sign` options.

The arm process is one of
1. Create the full set of artifacts on an arm machine (macbook, cloud vm, ...)
2. Use the ant build to copy and rename the `.tar.gz` with the native binaries only
3. Create a new `.asc `file.
4. Generate new sha512 checksum file containing the new name.
5. Move these files into the `downloads/release/$RC` dir

To perform these stages, you need a clean directory of the same
hadoop commit ID as for the x86 release.

In `build.properties` declare its location

```properties
arm.hadoop.dir=/Users/stevel/hadoop/release/hadoop
```

In that dir, create the relese.

```bash
time dev-support/bin/create-release --docker --dockercache --mvnargs="-Dhttp.keepAlive=false -Dmaven.wagon.http.pool=false" --deploy --native --sign
```

*Important* make sure there is no duplicate staged hadoop repo in nexus.
If there is: drop and restart the x86 release process to make sure it is the one published


```bash
# create the release.
# Broken until someone fixes HADOOP-18664. you can't launch create-release --docker from a build file
#ant arm.create.release
# copy the artifacts to this project's target/ dir, renaming
ant arm.copy.artifacts
# sign artifacts then move to the shared RC dir alongside the x86 artifacts
ant arm.release release.dir.check
```


### verify gpg signing

```bash
ant gpg.keys gpg.verify
```

### copy to a staging location in the hadoop SVN repository.

When committed to subversion it will be uploaded and accessible via a
https://svn.apache.org URL.


This makes it visible to others via the apache svn site, but it
is not mirrored yet.

When the RC is released, an `svn move` operation can promote it
directly.


*do this after preparing the arm64 binaries*

Final review of the release files
```bash
ant release.dir.check
```

Now stage the files, first by copying the dir of release artifacts
into the svn-mananaged location
```bash
ant stage
```

### In the staging svn repo, update, add and commit the work

This can take a while...exit any VPN for extra speed.


```bash
ant stage-to-svn
```

Manual
```bash
cd $stagingdir
svn update
svn add <RC directory name>
svn commit
```


### tag the rc and push to github

This isn't automated as it needs to be done in the source tree.

The ant `print-tag-command` prints the command needed to create and sign
a tag.

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

# How to download and build a staged release candidate

In build properties, declare `hadoop.version`, `rc` and `http.source`

```properties
hadoop.version=3.3.5
rc=3
http.source=https://dist.apache.org/repos/dist/dev/hadoop/hadoop-${hadoop.version}-RC${rc}/
```

### Targets of Relevance

| target               | action                     |
|----------------------|----------------------------|
| `release.fetch.http` | fetch artifacts            |
| `release.dir.check`  | verify release dir exists  |
| `release.src.untar`  | untar retrieved artifacts  |
| `release.src.build`  | build the source           |
| `release.src.test`   | build and test the source  |
| `gpg.keys`           | import the hadoop KEYS     |
| `gpg.verify`         | verify the D/L'd artifacts |
|                      |                            |

set `check.native.binaries` to false to skip native binary checks on platforms without them

### Download the RC files from the http server

Downloads under `downloads/incoming`
```bash
ant release.fetch.http
```

### untar source and build.

This puts the built artifacts into the local maven repo so
do not do this while building/testing downstream projects
*and call `ant purge-from-maven` after*

```bash
ant release.src.untar release.src.build
```


### untar site and build.


```bash
ant release.site.untar
```


### untar binary release

Untars the (already downloaded) binary tar to `target/bin-untar`

```bash
ant release.bin.untar
```

once expanded, the binary commands can be tested


```bash
ant release.bin.commands
```

This will fail on a platform where the native binaries don't load,
unless the checknative command has been disabled.
```properties
check.native.binaries=false
```

```bash
ant release.bin.commands -Dcheck.native.binaries=false
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

Download the artifacts from maven staging repositories and compile/test a minimal application

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

* This is java 11+ only.
* currently only builds against AWS v1 SDK.

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

And to to run the hadoop-cloud tests

```bash
ant spark.test.hadoop-cloud
```

A full spark test run takes so long that CI infrastructure should be used.

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


The test run is fairly tricky to get running; don't try and do this while
* MUST be java 11+
* Must have `cloud.test.configuration.file` set to an XML conf file
  declaring the auth credentials and stores to use for the target object stores
  (s3a, abfs, gcs)


## HBase filesystem

[hbase-filesystem](https://github.com/apache/hbase-filesystem.git)

Adds zookeeper-based locking on those filesystem API calls for which
atomic access is required.

Integration tests will go through S3A connector.

```bash
ant hboss.build
```

# After the Vote Succeeds: publishing the release

## Update the announcement and create site/email notifications

Edit `src/text/announcement.txt` to have an up-to-date
description of the release.

The `release.site.announcement` target will generate these
annoucements. Execute the target and then review
the generated files in `target/`

```bash
ant release.site.announcement
```

The announcement must be geneated before the next stage,
so make sure the common body of the site and email
annoucement is up to date: `src/text/core-announcement.txt`

## Build the Hadoop site

Set `hadoop.site.dir` to be the path of the
local clone of the ASF site repository
https://gitbox.apache.org/repos/asf/hadoop-site.git

```properties
hadoop.site.dir=/Users/stevel/hadoop/release/hadoop-site
```

Prepare the site; this also demand-generates the release announcement

The site .tar.gz distributable is used for the site; this must already
have been downloaded. It must be untarred and copied under the
SCM-managed `${hadoop.site.dir}` repository, linked up
and then committed.

```bash
ant release.site.untar

ant release.site.docs
```

Review the announcement.

### Manually link the site current/stable symlinks to the new release

In the hadoop site dir content/docs subdir

```bash

# update
git pull

# review current status
ls -l

# symlink current
rm current3
ln -s r.3.3.5 current3

# symlink stable
rm stable3
ln -s r3.3.5 stable3

# review new status
ls -l
```


Finally, *commit*

```bash
git add .
git status
git commit -S -m "HADOOP-18470. Release Hadoop 3.3.5"
git push
```


## Promoting the RC artifacts to production through `svn move`

```bash

# check that the source and dest URLs are good
ant staging-init
# do the promotion
ant stage-move-to-production
```

## update the `current` ref

```bash
https://dist.apache.org/repos/dist/release/hadoop/common
change the 
```
Check that release URL in your browser.

## Publish nexus artifacts

do this at [https://repository.apache.org/#stagingRepositories](https://repository.apache.org/#stagingRepositories)

to verify this is visible
[search for hadoop-common](https://repository.apache.org/#nexus-search;quick~hadoop-common)
-verify the latest version is in the production repository.

## Send that email announcement


## tag the final release and push that tag

The ant `print-tag-command` prints the command needed to create and sign
a tag.

```bash
ant print-tag-command
```

Use the "tagging the final release" commands printed

## clean up your local system

For safety, purge your maven repo of all versions of the release, so
as to guarantee that everything comes from the production store.

```bash
ant purge-from-maven
```
# tips

## Git status prompt issues in fish

There are a lot of files, and if your shell has a prompt which shoes the git repo state, scanning can take a long time.
Disable it, such as for fish:

```fish
set -e __fish_git_prompt_showdirtystate
```

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

# Rolling back an RC

Drop the staged artifacts from nexus
 [https://repository.apache.org/#stagingRepositories](https://repository.apache.org/#stagingRepositories)

Delete the tag. Print out the delete command and then copy/paste it into a terminal in the hadoop repo

```bash
ant print-tag-command
```

Remove downloaded files and maven artifactgs

```bash
ant clean purge-from-maven
```


1. Go to the svn staging dir
2. `svn rm` the RC subdir
3. `svn commit -m "rollback RC"`

```bash
ant stage-svn-rollback
# and get the log
ant stage-svn-log
```
