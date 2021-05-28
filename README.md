openSUSE Container Build Checks ![Hopefully green test status](https://github.com/Vogtinator/container-build-checks/actions/workflows/test.yml/badge.svg?branch=master)
===============================

This tool checks that built container images conform to the openSUSE container image policies (https://en.opensuse.org/Building_derived_containers).

It is designed to run at the end of image builds inside OBS, where it will look at all built images (both KIWI and DOCKER types) and fail the build if fatal issues were found. `make install` copies it to the directory where OBS expects it (https://github.com/openSUSE/obs-build/pull/675).

As input, it reads .containerinfo files generated by OBS, which contain information like the disturl, chosen tags and the name of the tarball in `docker save` format (other container tools are available). The latter contains `manifest.json`, the image config and layers.

To run it against built containers outside of OBS, it's necessary to craft a .containerinfo file containing the needed metainfo yourself, e.g.:

```
{
   "disturl" : "obs://build.opensuse.org/home:favogt:container-checks/containerfile/e6f02b3d61214e205f11d1c2075b14cc-dockerfile-application-container",
   "file" : "proper-derived.tar",
   "release" : "7.21",
   "tags" : [
      "opensuse/example:latest",
      "opensuse/example:5.7",
      "opensuse/example:5.7.7.21"
   ]
}
```

When called outside of OBS, it looks at all `*.containerinfo` files in the current directory.

Configuration
=============

Configuration files are in ini-format. All `*.conf` files in the configuration directory are read in alphabetical order. By default it looks for files in `/usr/share/container-build-checks/`, but the `CBC_CONFIG_DIR` environment variable can be used to override the path.

Available configuration options and their default values are:

```
[General]
# If true, treat all warnings as fatal. Errors are always fatal.
FatalWarnings=false
# If set, verify that the image-specific label prefix starts with this
Vendor=
# If set, verify that the org.opensuse.reference value uses this as registry
Registry=

[Tags]
# All tags have to match one of the patterns, e.g. opensuse/*,kubic/*
Allowed+=
# Tags must not match any of the patterns in here.
Blocked+=
```

Testing
=======

To build and run tests, run `make test` (parallel jobs are fine). If there are differences to the expected output, the diff will be shown and the test run fails.

Each subdirectory in `tests/` contains a `Dockerfile` used to build the tarball, a suitable `.containerinfo` with the metainformation and a `checks.out` file. The latter contains the expected output and exit status of running `container-build-checks.py` against that image.

Whenever the `Dockerfile` changes (tracked by the `built` stamp file), the image is rebuilt and the tarball updated. The image is tagged as `c-b-c-tests/<dirname>` and can be used as base image by other tests.

To update the `checks.out` files for all tests, run `make test-regen`.

`make lint` invokes `flake8` to perform linting on the source code.

Creating test cases
===================

To create a new test case, create a subdirectory in `tests/` and create or copy suitable `Dockerfile` and `*.containerinfo` files.

LABEL statements can be created from an existing image by running:

`podman image inspect foo | jq '.[0]["Config"]["Labels"]' | sed 's/": /=/g;s/^  "/LABEL /g;s/,$//g'`

Run `make test` (or just `make tests/foo/tested`) to see the output and `make test-regen` (or just `make tests/foo/regen`) to save it as `checks.out`.
