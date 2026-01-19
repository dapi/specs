# SRS-003 Application Versioning

This document describes the approaches to versioning developed applications in our products.

We separate the application version (the code) and the version of the built image that this code represents.

# Application Version (App Version)

The application version is formatted according to [semver 2.0](https://semver.org). Please note that we are talking not only about the version format, but also about the meaning of the major, minor, and patch parts of the version.

The application version is stored in the repository code in the source code.

The application version is updated by the developer manually or automatically at the time of the application release.

Example: `v1.2.3`

# Image Version (docker image tag)

This is the tag used to label the docker image. The tag format matches semver, only without the `v` prefix at the beginning.

Usually the image tag matches the application version, but may differ, as it is assigned to the image after building.

The same version of an application can be built through different pipelines, which will assign different tags to the resulting image.

For example, we have an application version `v1.2.3`, but we built a sub-image with only one part of the application and got `1.2.3-webserver`
