# SRS-012 Stand-Independent Images

What is described here applies to front-end applications as of 12/17/2025

The words `build` (as a result), `image`, `package` have the same meaning here and below.

The word `build` is used in two meanings: as the build process and as the build result (image, package).

# Motivation

Currently, for each stand, an individual build of the application must be performed, during which configuration specific to a particular stand is embedded into the application through environment variables.

This leads to the following:

* Blocks the ability to switch to deployment via **tagged builds**
* No ability to automatically rollback to the PREVIOUS version of a specific application, the entire stand, or the entire infrastructure.
* No ability to redeploy the current version of the application with new configuration/settings.
* No ability to redeploy the same code with different configuration, as the repository stores both pipeline changes and the code itself.
* The build and deployment process represents a single process, which deprives the ability to do them separately. To deploy an application of the required version, it needs to be built, which requires a stand for building and doesn't guarantee the same result.
* No centralized configuration management. To change a variable value (e.g., node address), you need to find all repositories where it's used, change the value in each one, rebuild and redeploy each application. Expected mechanism: change configuration in one place (infrastructure repository) and deploy to required stands with a single command.
* Violates 12factor in sections https://12factor.net/config and https://12factor.net/build-release-run (which leads to the problems described here)

## How build and deployment currently works

<img width="1416" alt="Screenshot 2024-12-17 at 08 43 48" src="https://github.com/user-attachments/assets/fc5d3a2c-b57c-48e0-81ac-4a47e78865db" />

## What is proposed to be done

1. Separate the build and deployment process.
2. Build applications into unified versioned packages (**tagged builds**). In other words, packages that have a version number specified in the package name or attached to it as a tag, as well as an application version that is returned by the application upon request or displayed in the UI.
3. During deployment, deploy the package from the package registry.
4. When deploying to production, deploy the package version that has been tested on the preliminary stands.

### Target build process

<img width="1142" alt="Screenshot 2024-12-17 at 09 05 26" src="https://github.com/user-attachments/assets/2a0ce6ab-e2be-4533-9150-00990af4ca98" />

### Target deployment process

<img width="917" alt="Screenshot 2024-12-17 at 09 05 41" src="https://github.com/user-attachments/assets/0f22d859-bde3-4034-8aed-01e538550a6b" />

## What are "Tagged builds"

A tagged build is a built package/image of an application that is stored in a registry (registry) and has a package version specified in the package name or attached to it as a tag, as well as an application version that is returned by the application upon request or displayed in the UI.

In this edition, a tagged build is a unified versioned application package that can be deployed to any stand.

1. Tagged builds allow improving the testing process. They allow confidence that production has exactly the version of the application that has been tested.
2. Have the ability to redeploy the application with a new configuration WITHOUT rebuilding (and consequently without retesting).
3. Allow uniquely identifying the version of the application on the stands. Currently, this is 5 stands and 4 applications = 20 versions.
4. Allow centralized management of all stands. In turn, this allows controlling the state of the infrastructure as a whole, having a backup through which the entire infrastructure can be brought to the target state with 2-3 commands.

## List of stand-dependent variables

```
VITE_APP_ENV: production

VITE_CF_ACCESS_CLIENT_ID:, VITE_CF_ACCESS_CLIENT_SECRET:

VITE_WC_PROJECT_ID:, APP_CONFIG_WC_PROJECT_ID:

VITE_BOT_SHARE_LINK:

VITE_BACKEND_URL:, APP_CONFIG_BACKEND_URL:, VITE_TOKEN_MONIT_BACKEND_URL, VITE_TOKEN_MONIT_WS_ENDPOINT:

APP_CONFIG_ENCODED_NODES_LIST:, VITE_NODE_URL_ETH:, VITE_NODE_URL_BSC:, VITE_NODE_URL_OPTIMISM:, VITE_NODE_URL_AVAX:, VITE_NODE_URL_ARBITRUM:, VITE_NODE_URL_POLYGON:
```

# Solution options

1. Embed ALL possible variable values into the image and select the needed one depending on the running stand, which is determined by the domain.
2. Pull the necessary values via `fetch()` from the specified URL from the running JS application.
3. Set the necessary values through a JS file that is connected in parallel in index.html. `<script type="module" src="/config.js"></script>`. In other words, configs are set via es6 module.
4. Have a build with stub values that are replaced with stand-dependent ones:
   - (a) during deployment process
   - (b) via ngx_http_sub_module and similar solutions.
5. Embed values through meta-headers in index.html.
   - (a) During deployment through substitutions.
   - (b) By a separate process to be placed on stands in advance.
6. Use domain binding to determine URLs for api, ws, and nodes.
7. Others?

# Weighing options

[<img width="1886" alt="Screenshot 2024-12-17 at 13 16 46" src="https://github.com/user-attachments/assets/2c82e8e7-b9c7-44df-a691-ccdef5e90429" />](https://docs.google.com/spreadsheets/d/1hg1KAWH6n70XV_ryseQ8d85daG76vz2A8rcely_d0xc/edit?usp=sharing)


