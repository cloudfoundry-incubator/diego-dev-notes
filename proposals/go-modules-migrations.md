# BOSH release migration path to use Go modules

## Summary

After [proposing
different](https://docs.google.com/document/d/1MeiXIqzsj_j1ziYAfhVXfCCFJLiKhi7HiOyF3g-7Wpk/edit#heading=h.9jamy725425y)
routes for converting bosh releases to use Go modules, we are going to move
forward with the following implementation:

- Single go module under `src/code.cloudfoundry.org` with a generic name
  `code.cloudfoundry.org`

As far as estimation for this work, I estimate 1-2 weeks of work for
diego-release as an example of a release with over 130 submodules.

## Migration Path

Here are steps taken for conversion

```
cd ./src/code.cloudfoundry.org
go mod init code.cloudfoundry.org
go mod vendor -e #ignore errors
# run tests
# if tests failed due to mismatch on version, try the submoduled version instead
```

### Add Depedabot for independent modules (e.g. code.cloudfoundry.org/localip)

When a new module is created, let's also add dependabot so that the dependencies
can remain updated.
[archiver](https://github.com/cloudfoundry/archiver/blob/2762da2677ce6ba931a6e9fff947c7541f470615/.dependabot/config.yml)
is an example of the dependabot config.

### Add Github action integration for running the test for independent modules

When a new module is created, let's also add github action integration for
running the tests

### Module dependency versions

BOSH releases have specific SHAs checkout out for each dependency. [Some
repositories](https://github.com/cloudfoundry/diego-release/blob/68b60677acffd6ab241e2698f581c52f5da3ed83/.dependabot/config.yml)
are setup to receive security update (if dependabot security bump is working as
expected), but we are not bumping dependencies to the latest all the time.  When
these are converted to Go module's dependency:

- Go mod tooling will automatically get the latest one from the remote
  repository
- Ensure tests pass with the latest version
- If Broken, let's lock it back to the submoduled SHA and exclude that from
  dependabot


### Vendoring external modules that use replace directive

[`replace`
directive](https://thewebivore.com/using-replace-in-go-mod-to-point-to-your-local-module/)
is very handy when a developer is making changes to modules for BOSH releases
and wants to see the changes applied after deploy without having to commit the
change to upstream. Using `replace` directive works when it's internal to the
release, but when the component is used outside of the release (e.g.
`diego-release` uses `guardian` from `garden-runc-release`) we would have to
manually add the replace commands before being able to vendor `guardian`. In
order to use `guardian`, you must clone the needed repositories using submodules. Here
is an example workflow when adding those components

```
cd inigo
go mod init
go mod edit \
 -replace code.cloudfoundry.org/guardian=./guardian \
 -replace code.cloudfoundry.org/garden=./garden \
 -replace code.cloudfoundry.org/grootfs=./grootfs \
 -replace code.cloudfoundry.org/idmapper=./idmapper
go mod vendor
```

### Add a script to update golang version for each module

Each module would have a script to update go version. [More
info](https://golang.org/ref/mod#go-mod-file-go) for how to do that.


## Work-In-Progress

- Get all of the builds green with go 1.15.8 and go.mod changes. (last failure:
  inigo was failing to build guardian)
- Bump golang to 1.16.4
- Let the pipeline run everything with gomodules
- remove github.com/docker code under src which is probably not used anymore
  after conversion
- try to find any regression since we are running a fairly new version of diego.
  [Here is an example
  issue](https://github.com/cloudfoundry/dockerapplifecycle/issues/7). If we are
  incompatible we should either decided to be compatible or that our release
  notes should reflect the incompatibility.


## End Goal

After migration is completed:

- A release should have a single submodule
- `GOPATH` in `.envrc` is removed
- Release is using Go 1.16+
- `GO111MODULE` is unset
- When we `ag GOPATH`, all results should be converted/removed including docs
