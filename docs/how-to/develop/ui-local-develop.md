---
sidebar_position: 1
---

# feature-probe-ui

## Requirements

[Node.js](https://nodejs.org/en/download/) version 16.13 or above (which can be checked by running `node -v`). You can use [nvm](https://github.com/nvm-sh/nvm) for managing multiple Node versions on a single machine installed.

### Installation


```shell
git clone https://github.com/FeatureProbe/feature-probe-ui.git
cd feature-probe-ui
yarn install
```

:::tip
It is highly recommended that use `yarn` as the package manager.If you are using `npm install` to install and encountering this error as below, use `npm install --legacy-peer-deps` instead
:::
 
![toggles screenshot](/local_develop_install_error.png)

```shell
npm install --legacy-peer-deps
```

### Local Development

```
yarn start
```

This command starts a local development server and opens up a browser window, the default port is 3000. Visit `http://localhost:3000` to get the UI page.


### Proxying API Requests in Development

By adding a proxy filed in `package.json`, you can tell the development server to proxy all the requests to an API server in development.

```
"proxy": "https://featureprobe.io/"
```

The default setting is our online demo service: `https://featureprobe.io/`, if you have deployed your own API service, change this field your own.


### Deloyment

If you want to deploy UI project independently, refer this doc: [UI Service](../deploy/deployment-source-code#ui-%E6%9C%8D%E5%8A%A1)