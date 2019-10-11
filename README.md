Crontinuous Build System
========================

Crontinuous Build is pull-only 'continuous' build system, designed to build
projects from various git sources (including github) without having to present
any public interface.

The system is divided into 4 steps:

 - Acquire -- identify repositories that can be build
 - Poll    -- scan repositories for changes
 - Build   -- download a tarball and run the build rules
 - Report  -- report build status to any endpoints

Build Rules
-----------

Crontinuous Build expects that repositories define most of their own build
procedure. However, a basic rules system to determine how to trigger builds
is included configured in files in `/etc/crontinuous/rules.d/`.

These take the format of one directive per line; currently three directives
These take the format of:

| Directive         | Argument        | Description                                                          |
| ----------------- | --------------- | -------------------------------------------------------------------- |
| `FileExists`      | `string`        | Checks if a file exists (no glob)                                    |
| `DirectoryExists` | `string`        | Checks if a directory exists (no glob)                               |
| `BuildBranches`   | `string`        | Space separated list of names/globs of branches that should be built |
| `BuildTags`       | `string`        | Space separated list of names/globs of tags that should be built     |
| `Environment`     | `string=string` | Set an environment variable for build                                |
| `EnvironmentCMD`  | `string=string` | Set an environment variable for build based on a command             |
| `Step`            | `string`        | Run a command as part of the build                                   |

Additionally, the following environment variables are set:

| Variable          | Example         | Description                           |
| ----------------- | --------------  | ------------------------------------- |
| `REPO_URL`        | `git@github…`   | The URL of the repo being built       |
| `COMMIT_HASH`     | `ab4663f`       | 'Short' (7 digit) commit hash         |
| `BRANCH`          | `master`        | The branch (or other head)            |
| `TAG`             | `blah`, `1.1-1` | The nearest tags to the commit        |

For example, the default build rules for use of
[https://github.com/javajawa/bpkg|bpkg] is:

```
DirectoryExists=debian
FileExists=Makefile

BuildBranches=*
BuildTags=*

EnvironmentCMD=VERSION=test "$TAG" = "$BRANCH" && printf "%s" "$TAG" || printf "%s~%s" "$TAG" "$COMMIT_HASH"
Step=bpkg-build . "$VERSION"
```

Acquire
-------

"Acquisition" is the stage in which the list of repositories to build
is updated. A repository is considered 'buildable' if it matches one
of the build rules in `/etc/crontinous/rules.d/` (see Build Rules).

Each helper configured in `/etc/crontinous/helpers.conf` is called with the
`acquire` argument, which returns the URLs to each buildable repository.
Each repository is given a unique ID, and a file is created in
`/etc/crontinuous/repositories.d/`.

Acquisition can occur either on the systemd timer `crontinuous-acquire.timer`
or manually.

Poll + Build
------------

Pooling triggered by `crontinuous-build.timer`; each repository is scanned
with the use of its helper, and any branches or tags that match the
build rules are queued to be downloaded and built.
Builds are run as one-shot services running within `crontinuous.slice`,
allowing them to be limited by cgroups.

The build psuedo-service will use the helper to download a tarball of the
git commit to be built to a temporary directory, set up the environment
and then run through the build steps.

The service uses systemd's folder protection to make sure the build can only
write to the build directory, and `/tmp` in general. This does not fully protect
the file system, but reduces the potential for unexpected side effects.

Report
------

The design for reporting has not yet been confirmed.
