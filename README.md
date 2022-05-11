# Validate Hadoop Client Artifacts

This project imports the hadoop client artifacts to verify that they are (a) published on the maven repository and (b) contain the classes we expect.

It also has an ant `build.xml` file to help with preparing the release,
validating gpg signatures, creating release messages and other things.


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


