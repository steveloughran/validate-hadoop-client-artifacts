# Validate Hadoop Client Artifacts

This project imports the hadoop client artifacts to verify that they are (a) published on the maven repository and (b) contain the classes we expect.

To build and test with the client API:l

```bash
mvn clean test -Pclient
```

Compilation verifies the API is present; the
test verifies that some shaded artifacts are present.

If the hadoop artifacts are in staging/snapshot repositories,
use the `staging` profile

```bash
mvn clean test -Pclient -Pstaging
```

To force an update

```bash
mvn clean test -Pclient -Pstaging -U
```

To purge all artifacts of the chosen hadoop version from your local maven repository.

```bash
mvn clean -Ppurge
```

*do not use with the dependency declaration of -Pclient or things get confused*
(it will try to resolve the artifacts, even as they are deleted)

There are different profiles for different versions;
the default value is 3.3.3
