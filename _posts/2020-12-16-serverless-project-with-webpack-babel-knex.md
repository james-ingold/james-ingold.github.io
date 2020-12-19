---
title: Setting up a Serverless Project with Webpack, Babel, and Knex
description: Fix common issues with Serverless and Knex
author: James Ingold
published: true
---

Using Webpack with the Serverless Framework is handy if you want to use the latest Javascript features along with Babel. It also helps to optimize the packaging of functions so we can make sure we're only shipping code that is lean and mean. However, adding the delightful query builder Knex to the mix can cause some issues that I spent a good amount of time on. Hopefully this article will help anyone dealing with similar issues save some time.

In this article we'll go through setting up a Serverless project with Webpack, Babel, and Knex along with Prettier and Eslint. We'll focus on specific issues with Knex in this scenario and how to solve them. If you want a TLDR; here's the final output, a [Serverless starter template](https://github.com/james-ingold/serverless-webpack-babel-knex-starter) with Webpack, Babel, and Knex ready to go.

#### Project Setup

Install serverless globally

```
npm install serverless -g
```

First, we'll set up a new Serverless project using a default aws-nodejs template:

```
serverless create --template aws-nodejs
```

This will create a bare handler.js and a serverless yaml file to get us started.

Next add a package.json file to manage our dependencies

```
npm init -y
```

#### Add Dev Dependencies and Webpack:

We're going to add Babel to get access to the latest Javascript features and then we'll add Webpack to transform our Javascript code in a way that the Serverless platforms (AWS) can handle. We'll also add Serverless-Offline which emulates AWS and AWS Gateway, allowing us to run our functions locally.

```
npm install --save-dev @babel/core @babel/preset-env webpack serverless-webpack serverless-offline babel-loader dotenv
```

#### Adding Source Map Support

It's always nice to get stack traces, let's set up source map support.

```bash
npm install source-map-support --save npm install
babel-plugin-source-map-support --save-dev
```

The source-map-support module provides source map support for stack traces in node via the [V8 stack trace API](https://www.npmjs.com/package/source-map-support)

Babel-plugin-source-map-support prepends this statement to each file, giving us stack traces with the source-map-support package:  
`import 'source-map-support/register';`

#### Setting up Babel

Create a .babelrc file in the root of the project to handle our Babel configuration:

.babelrc

```json
{
  "plugins": ["source-map-support"],
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "node": true
        }
      }
    ]
  ]
}
```

#### Adding Knex

Next, we'll add Knex and MySQL as the driver of choice for this purpose:

```
npm install --save mysql2 knex
```

#### Setting up Knex

Create a knexfile.js in the project root:

```js
import dotenv from "dotenv";
dotenv.config({ silent: true });

module.exports = {
  development: {
    client: "mysql2",
    connection: {
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PW,
      database: process.env.DB_DB
    }
    // migrations: {
    // directory: './database/migrations',
    // },
    // seeds: { directory: './database/seeds' }
  },
  staging: {
    client: "mysql2",
    connection: {
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PW,
      database: process.env.DB_DB
    }
  },
  production: {
    client: "mysql2",
    connection: {
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PW,
      database: process.env.DB_DB
    }
  }
};
```

Create a folder called queries in your project root, this will be where the data retrieval functions will go:

```
mkdir queries
```

Add a knex file:
knex.js

```js
const knex = require("knex");

const knexfile = require("../knexfile");

const env = process.env.NODE_ENV || "development";

const configOptions = knexfile[env];

module.exports = knex(configOptions);
```

Example query file - games.js:

```js
const knex = require("./knex");

export async function getAll() {
  const res = await knex("matches").select("*");
  return res;
}
```

#### Setting up Webpack

In the root of the project create a webpack.config.js file and configure Webpack to use Babel to bundle up our Serverless functions.
We'll also exclude node development dependencies using the node externals package.

```
npm install webpack-node-externals --save-dev
```

webpack.config.js:

```js
const slsw = require("serverless-webpack");
const nodeExternals = require("webpack-node-externals");

module.exports = {
  entry: slsw.lib.entries,
  devtool: "source-map",
  // Since 'aws-sdk' is not compatible with webpack,
  // we exclude all node dependencies
  externalsPresets: { node: true },
  externals: [nodeExternals()],
  mode: slsw.lib.webpack.isLocal ? "development" : "production",
  optimization: {
    minimize: false
  },
  performance: {
    // Turn off size warnings for entry points
    hints: false
  },
  // Run babel on all .js files - skip those in node_modules
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: "babel-loader",
        include: __dirname,
        exclude: /node_modules/
      }
    ]
  },
  plugins: []
};
```

#### Setting up Serverless

Add our plugins to the serverless.yaml file:

```
- serverless-webpack
- serverless-offline
```

Add serverless-webpack configuration to serverless.yaml

```yaml
custom:
  webpack:
    webpackConfig: ./webpack.config.js
    includeModules: true # enable auto-packing of external modules
```

We'll add an http endpoint to the default hello handler, so that we can test our api endpoint out:

```yaml
events:
  - http:
    path: hello
    method: get
    cors: true
```

Full Serverless.yaml

```yaml
service: serverless-webpack-babel-knex-starter
frameworkVersion: "2"

provider:
name: aws
runtime: nodejs12.x
apiGateway:
  shouldStartNameWithService: true

plugins:

- serverless-webpack
- serverless-offline

functions:
  hello:
    handler: handler.hello
      events:
        - http:
          path: hello
          method: get
          cors: true

custom:
  webpack:
  webpackConfig: ./webpack.config.js
  includeModules: true # enable auto-packing of external modules
```

#### Running and Knex Issues

Let's test it out!
Add a start npm script to package.json

```json
"start": "serverless offline start --stage dev --noAuth"
```

Call our API

```bash
curl --location --request GET 'http://localhost:3000/dev/hello'
```

#### Knex Runtime Issues:

- ES Modules may not assign module.exports or exports.\*, Use ESM export syntax, instead: ./knexfile.js

It doesn't like that we're using module.exports in our knexfile, one potential solution would be to use es6 default export syntax
export default {}

This ended up causing more problems then it solved dealing with the internal knex library which doesn't play well with ES modules.

The solution I went for is to use a Babel plugin to transform ESM to CommonJS Modules which is the standard for Node modules. Client-side JavaScript that runs in the browser use another standard, called ES Modules or ESM.
In CommonJS, we export with module.exports and import with require statements. Since we're using Babel we can use import/export and our code will be transformed into CommonJS modules.

```
npm install --save-dev @babel/plugin-transform-modules-commonjs
```

Add to our plugins section in .babelrc

```json
{
  "plugins": ["source-map-support", "@babel/plugin-transform-modules-commonjs"],
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "node": true
        }
      }
    ]
  ]
}
```

Using CommonJS should be enough to get you going but you might run into the next issue:

- Can't resolve runtime dependencies

```
Module not found: Error: Can't resolve 'oracledb'
Module not found: Error: Can't resolve 'pg-native'
Module not found: Error: Can't resolve 'pg-query-stream'
Module not found: Error: Can't resolve 'sqlite3'
```

If you receive module not found errors for packages that you are not using, then we can fix this by ignoring those drivers/packages.
There are different ways this can be approached with Webpack and with Serverless but the solution that I landed on was to use the NormalModuleReplacementPlugin which is bundled with Webpack. This plugin allows you to replace resources that match a regular expression with another resource. We'll add the noop2 package to replace the drivers we're not using with a "no operation module".

```
npm install --save-dev noop2
```

```js
const { NormalModuleReplacementPlugin } = require("webpack");

plugins: [
  // Ignore knex runtime drivers that we don't use
  new NormalModuleReplacementPlugin(
    /mssql?|oracle(db)?|sqlite3|pg-(native|query)/,
    "noop2"
  )
];
```

#### Adding Eslint and Prettier

To finish this starter template, we'll add some niceness to the project with eslint and prettier.

```
npm install --save-dev @babel/eslint-parser eslint eslint-config-prettier eslint-plugin-lodash eslint-plugin-prettier prettier
```

prettierrc.json

```json
{
  "trailingComma": "none",
  "tabWidth": 2,
  "semi": false,
  "singleQuote": true,
  "printWidth": 120
}
```

.eslintrc.js

```js
module.exports = {
  env: {
    node: true
  },
  plugins: ["prettier"],
  parser: "@babel/eslint-parser",
  parserOptions: {
    sourceType: "module",
    ecmaFeatures: {
      classes: true,
      experimentalObjectRestSpread: true
    }
  },
  extends: [
    "eslint:recommended",
    "plugin:prettier/recommended",
    "plugin:lodash/recommended"
  ],
  rules: {
    "prettier/prettier": "error"
  }
};
```

#### Starter Project

Now we have a nice starter project to get us off the ground with Serverless, Webpack, Babel, and Knex.

To grab all this goodness or if you have improvements, check out the [Github
repository](https://github.com/james-ingold/serverless-webpack-babel-knex-starter)
