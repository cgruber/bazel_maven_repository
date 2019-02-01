# Bazel Rules for Maven Respositories

A bazel ruleset creating a more idiomatic bazel representation of a maven repo using a
pinned list of artifacts.

Release: `1.0-rc2`

| Link | Sha |
| ---- | --- |
| [Zip File](https://github.com/square/bazel_maven_repository/archive/1.0-rc2.zip) | `db73debcdd6b1a593e9ff44932fbeaf5c9b00ec829fa02cd00b646fe53b8e8ba` |
| [Tarball](https://github.com/square/bazel_maven_repository/archive/1.0-rc2.tar.gz) | `b8173f4d6c0c39cb13c259d8713f84051b529b4475cef04e8ee3d74cb26a30e7` |


## Overview

**Bazel Rules for Maven Repositories** allow the specification of a list of artifacts which
constitute maven repository's universe of deps, and exposes these deps into a bazel *repository*
namespace.  The name of the repository specification rule becomes the repository name in Bazel.
For instance the following specification:
 
```python
MAVEN_REPOSITORY_RULES_VERSION = "1.0-rc2"
MAVEN_REPOSITORY_RULES_SHA = "db73debcdd6b1a593e9ff44932fbeaf5c9b00ec829fa02cd00b646fe53b8e8ba"
http_archive(
    name = "maven_repository_rules",
    urls = ["https://github.com/square/bazel_maven_repository/archive/%s.zip" % MAVEN_REPOSITORY_RULES_VERSION],
    type = "zip",
    strip_prefix = "bazel_maven_repository-%s" % MAVEN_REPOSITORY_RULES_VERSION,
    sha256 = MAVEN_REPOSITORY_RULES_SHA,
)
load("@maven_repository_rules//maven:maven.bzl", "maven_repository_specification")
maven_repository_specification(
    name = "maven",
    artifacts = {
        "com.google.guava:guava:25.0-jre": { "sha256": "3fd4341776428c7e0e5c18a7c10de129475b69ab9d30aeafbb5c277bb6074fa9" },
    }
)
```
 
... results in deps of the format `@maven//com/google/guava:guava` (which can be abbreviated to 
`@maven//com/google/guava`)

Dependency versions are resolved in the single artifact list.  Only one version is permitted within
a repository.

> Note: bazel_maven_repository has no workspace dependencies, so adding it to your project will not
> result in any additional bazel repositories to be fetched.

## Supported Types

Currently `.aar` and `.jar` artifacts are supported.  OSGI bundles are supported by assuming they are
normal `.jar` artifacts (which they are, just have a packaging property of `bundle` and some extra
metadata in `META-INF` of the `.jar` file).

`.aar` artifacts should be specified as `"some.group:some-artifact:1.0:aar"` (just append `:aar`
onto the artifact spec string). 

For any other types, please file a feature request, or supply a pull request.  So long as there
exists a proper bazel import or library rule to bring the artifact's file into bazel's dependency
graph, it should be possible to support it.

## Inter-artifact dependencies

This rule will, in the generated repository, infer inter-artifact dependencies from pom.xml files
of those artifacts (pulling in only `compile` and `runtime` dependencies, and avoiding any `systemPath`
dependencies).  This avoids the bazel user having to over-specify the full set of dependency jars.

All artifacts, even transitively depended-on ones, need to be specified with pinned versions in the
`artifacts` property, and any artifacts discovered in the inferred dependency search, which are not
present in the main rule's artifact list will be flagged and the build will fail with an error listing
them.

## Coordinate Translation

Translation from maven group/artifact coordinates to bazel package/target coordinates is naive but
orderly.  The logic mirrors the layout of a maven repository, with group_id elements (separated by
`.`) turning into a package hierarchy, and the artifact_id turning into a bazel target. 

### Mangling

Bazel tends not to like package and target names using anything other than `[A-Za-z9-0_]` (though it
can support dashes in some cases).  These rules do a straight mangling of other characters into `_`
in artifact_ids (though not in group_ids because: reasons).

While this typically turns into what you'd expect, there are a few times where it doesn't. 

For instance:

```python
maven_repository_specification(
    name = "maven",
    insecure_artifacts = [
        "org.mockito:mockito-core:1.9.5",
        "joda-time:joda-time:1.1",
    ],
)
```
 
would be referenced in a rule like so:

```python
java_library(
    name = "foo",
    srcs = glob(["*.java"]),
    deps = [
        "@maven//org/mockito:mockito_core",
        "@maven//joda-time:joda_time",
    ],
)
```

> Note: The package/workspace layout generated by the `maven_repository_specification` rule can be
> found at `<workspace>/bazel-<workspace_name>/external/<maven_repo_name>` (all bazel generated
> workspaces are available in `bazel-yourworkspace/external`).  The package structure can be
> inspected if it is confusing.

## Artifact Configuration
### Sha verification

Artifacts with SHA256 checksums can be added to `artifacts`, in the form:
```
    artifacts = {
        "com.google.guava:guava:25.0-jre": { "sha256": "3fd4341776428c7e0e5c18a7c10de129475b69ab9d30aeafbb5c277bb6074fa9" },
    }
```
Artifacts without SHA headers should configured as insecure, like so:
```
    artifacts = {
        "com.google.guava:guava:25.0-jre": { "insecure": True },
    }
```

The rules will reject artifacts without SHAs are not marked as "insecure". 

> Note: These rules cannot validate that the checksum is the right one, only that the one supplied
> in configuration matches the checksum of the file downloaded.  It is the responsibility of the
> maintainer to use proper security practices and obtain the expected checksum from a trusted source.

### Substitution of build targets

One can provide a `BUILD.bazel` target snippet that will be substituted for the auto-generated target
implied by a maven artifact.  This is very useful for providing an annotation-processor-exporting
alternative target.  The substitution is naive, so the string needs to be appropriate and any rules
need to be correct, contain the right dependencies, etc.  To aid that it's also possible to (on a
per-package basis) substitute dependencies on a given fully-qualified bazel target for another. 

A simple use-case would be to substitute a target name (e.g. "mockito-core" -> "mockito") for
cleaner/easier use in bazel:

```python
maven_repository_specification(
    name = "maven",
    artifacts = {
        "org.mockito:mockito-core:2.20.1": {
            "sha256": "blahblahblah",
            "build_snippet": """maven_jvm_artifact(name = "mockito", artifact = "org.mockito:mockito-core:2.20.1")""",
        },
        # ... all the other deps.
    },
)
```

This would allow the following use in a `BUILD.bazel` file.

```python
java_test(
  name = "MyTest",
  srcs = "MyTest.java",
  deps = [
    # ... other deps
    "@maven//org/mockito" # instead of "@maven//org/mockito:mockito-core"
  ],
)
```

More complex use-cases are possible, such as adding substitute targets with annotation processing `java_plugin`
targets and exports.  An example with Dagger would look like this (with the basic rule imports assumed):

```python
DAGGER_PROCESSOR_SNIPPET = """
# use this target
java_library(name = "dagger", exports = [":dagger_api"], exported_plugins = [":dagger_plugin"])

# alternatively-named import of the raw dagger library.
maven_jvm_artifact(name = "dagger_api", artifact = "com.google.dagger:dagger:2.20")

java_plugin(
    name = "dagger_plugin",
    processor_class = "dagger.internal.codegen.ComponentProcessor",
    generates_api = True,
    deps = [":dagger_compiler"],
)
"""
```

The above is given as a substitution in the `maven_repository_specification()` rule.  However, since the inferred
dependencies of `:dagger-compiler` would create a dependency cycle because it includes `:dagger` as a dep, the
specification rule also should include a `dependency_target_substitution`, to ensures that the inferred rules in
the generated `com/google/dagger/BUILD` file consume `:dagger_api` instead of the wrapper replacement target.

```python
maven_repository_specification(
    name = "maven",
    artifacts = {
        "com.google.dagger:dagger:2.20": {
            "sha256": "blahblahblah",
            "build_snippet": DAGGER_PROCESSOR_SNIPPET,
        },
        "com.google.dagger:dagger-compiler:2.20": { "sha256": "blahblahblah" },
        "com.google.dagger:dagger-producers:2.20": { "sha256": "blahblahblah" },
        "com.google.dagger:dagger-spi:2.20": { "sha256": "blahblahblah" },
        "com.google.code.findbugs:jsr305:3.0.2": { "sha256": "blahblahblah" },
        # ... all the other deps.
    },
    dependency_target_substitutes = {
        "com.google.dagger": {"@maven//com/google/dagger:dagger": "@maven//com/google/dagger:dagger_api"},
    }
)
```

Thereafter, any target with a dependency on (in this example) `@maven//com/google/dagger` will invoke annotation
processing and generate any dagger-generated code.  The same pattern could be used for
[Dagger](http://github.com/google/dagger), [AutoFactory and AutoValue](http://github.com/google/auto), etc.

Such snippet constants can be extracted into .bzl files and imported to keep the WORKSPACE file tidy. In the
future some standard templates may be offered by this project, but not until deps validation is available, as
it would be too easy to have templates' deps lists go out of date as versions bumped, if no other validation
prevented it or notified about it.

### Packaging

Optionally, an artifact may specify a packaging. Valid artifact coordinates are listable this way:
`"group_id:artifact_id:version[:packaging]"`

At present, only `jar` (default) and `aar` packaging are supported.

### Classifiers

Classifiers have only limited support. An artifact can specify a classifier, but only that or
the unclassified artifact can be used, but not both.

Classifiers are tacked on the end, e.g. `"foo.bar:blah:1.0:jar:some-classifier"`
 
## Limitations

  * Doesn't support -SNAPSHOT dependencies, but will in a future version.
  * Doesn't support multiple versions of a dependency (by design).
  * Doesn't support multiple calls to `maven_repository_specification()` due to collisions in
    the implicit fetching rules it creates. This limitation will be lifted in a version.

## Other Usage Notes

Because of the nature of bazel repository/workspace operation, updating the list of artifacts may
invalidate build caches, and force a re-run of workspace operations (and possibly reduce
incrementality of the next build).  This is unavoidable.

It may make sense, if one's maven universe gets big, to extract the list of artifacts into a 
constant in a separate file (e.g. `maven_artifacts.bzl`) and import it. 

# License
License
Copyright 2018 Square, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
