# Validate Hadoop Client Artifacts

This project imports the hadoop client artifacts to verify that they are (a) published on the maven repository and (b) contain the classes we expect.

It has an ant `build.xml` file to help with preparing the release,
validating gpg signatures, creating release messages and other things.

# ant builds

Look in the build.xml for details, including working with other modules





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

# download and build someone else's release candidate

In build properties, declare `hadoop.version`, `rc` and `http.source`

```properties
hadoop version=2.10.2
rc=0
http.source=https://home.apache.org/~iwasakims/hadoop-2.10.2-RC0/
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
|                    |                            |
|                    |                            |
|                    |                            |
|                    |                            |

set `release.native.binaries` to false to skip native binary checks on platforms without them
