---
title: Gulp/Bower => Browserify => Webpack. An AngularJS Journey
description: Transitioning an AngularJS from a Gulp/Bower build process to Webpack
author: James Ingold
published: true
---

A Gulp/Bower post in 2020? I know, but when you've been heads down working in a new industry, getting feedback, building and pivoting to keep up with the changing landscape: things like your build process just don't seem that important. If it ain't broke, don't fix it. The time had come though, to transition a flagship AngularJS app off Gulp/Bower to Webpack.
Some background: In 2014, I had opportunity to select the frontend framework for what at the time was going to be a next generation Electronic Medical Record application. The choices were basically AngularJS, Durandal, Ember, and Backbone. React was a baby, about a year old. AngularJS was the hotness, a few years old and backed by Google. It also sported an intuitive Model-View-Controller framework that I knew developers on the team would be able to pick up (once they got past some AngularJS black magic and naming conventions). It has turned out to be a solid choice and has supported the development efforts well, for over six years. Allowing the team to move fast and keep up with changing stakeholder needs. However, its writing is on the wall and this is the first step to making a smooth transition.

### Motivations

- To be able to write the same version of Javascript on the frontend and the backend. Lessen context switching.
- Staying current with latest Javascript changes, return to cutting edge form. "When you spend all your time on features, the inevitable outcome is that easy tasks become difficult and take longer."
- To pave the way for slowly transitioning away from AngularJS
- Kaizen culture => everything around you can be improved and deserves to be improved

### The Process

I had actually attempted to make this change twice before, going from Gulp => Webpack. However, I had failed both times. This was a large scale change, I had to update the code to use ES Modules in AngularJS and write the Webpack configurations for both production and development. Current web frameworks come pre-rolled with the Webpack configuration (Angular CLI, Vue CLI, etc). You usually don't have to write your own Webpack configuration and even back in the early Vue days, you just had to modify a few bits for your production build process. Writing one from scratch for an already existing app is a tall order. Webpack introduces a new way of thinking with it's entry, output, loader and rules. It's definitely less intuitive than Gulp which is just passing streams around.

So for those first two attempts, I got hung up on Webpack. I spent a lot of time spinning my wheels. I had written a couple Webpack configurations before in greenfield projects and I had modified my fair share but moving an existing Gulp configuration to Webpack just wasn't clicking.

##### Enter Browserify.

require('modules') in the browser. I had never used Browersify before, I had heard of it but mainly in the context that it was Webpack's younger brother and you should just use Webpack.

Pros:

- Dead simple, command line first.
- Sticks to the Linux philosophy of do one thing well.

Cons:

- You're probably going to want more functionality in a complex application.
- Doing everything on command line can be hard for some developers to follow.
- The configuration options are not that great, I don't want to put browserify properties into package.json. It just feels wrong to me.

Browserify is Punk-Rock to Webpack's Top 40 Hits. Learning about Browserify just clicked and I started to devise a plan on getting this app bundled. I really enjoyed learning about Browserify, everything about the project resonated with me. Equipped with some Browersify knowledge, I could now move forward.

##### Implementation

Here's what I needed to do to move an AngularJS app from Gulp/Bower to Browersify:

1. Update AngularJS modules to ES modules. I wanted to keep as much of the codebase intact as possible and not harm any developers productivity. We use folder by feature/module structure and using the AngularJS module as the entry point was the best way to do this. This allows us to ESnext our javascript files more incrementally. For Browserify, I used bulk-require and bulkify (Browserify plugins all end in ify which is nice). Here's an example of ES Moduling a standard AngularJS module

   Before:

   ```js
   (function() {
     "use strict";
     angular.module("blocks.logger", []);
   })();
   ```

   After:

   ```javascript
   angular.module("blocks.logger", []); // create the module
   const bulk = require("bulk-require");
   // bulk require all the files in this folder such as logger.js
   bulk(__dirname, ["./**/!(*.module).js"]);
   export default angular.module("blocks.logger"); // export our module
   ```

2. Use app.js as the entry file and use import syntax to include all the modules and dependencies for the application.

   Before:

   ```javascript
   (function () {
     'use strict'

     var app = angular
       .module('app', [
         'common',
         'blocks.logger',
         'blocks.exception'
         ...etc
       ])
   ```

   After:

   ```javascript
   // globals - lodash, jquery, etc go here

   import angular from 'angular/index'
   // other angularjs depencies go here, ui-router, etc
   import ngRoute from 'angular-route'

   // application modules
   import logger from './blocks/logger/module'
   import common from './common/module'
   import exception from './blocks/logger/exception'

   var app = angular.module('app', [
       ngRoute,
       'blocks.exception',
       'blocks.logger',
   	...etc

   export default app
   ```

3. Move frontend dependencies from Bower into modules
   This is pretty simple, just npm install -s the dependencies you're using and import them in app.js.

   ```javascript
   import $ from jquery
   ```

4. Shim globals  
   For this app, there was existing code in the pug index file that relied on jQuery being on the window and AngularJS needs to pull in jQuery or it will use JQlite. For this, there is the shim-browersify plugin.  
    package.json

   ```json
   {
     "browser": {
       "angular": "./node_modules/angular/angular.js",
       "jquery": "./node_modules/jquery/dist/jquery.js"
     },
     "browserify": {
       "transform": ["browserify-shim"]
     },
     "browserify-shim": {
       "angular": {
         "depends": "jquery:jQuery",
         "exports": "angular"
       }
     }
   }
   ```

5. Browersify build script using [tinyify](https://github.com/browserify/tinyify){:target="\_blank"} for minification

   ```
      browserify -t [ babelify --presets [ @babel/preset-env ] ] -t bulkify public/app/app.js -o public/bundle.js -p [ tinyify --no-flat ]
   ```

6. Browerify dev script - enter [watchify](https://github.com/browserify/watchify){:target="\_blank"}. Watches files in bundle for changes and updates only what changed. Creates sourcemaps.

   ```
     watchify --full-paths -t [ babelify --presets [ @babel/preset-env ] ] -t bulkify public/app/app.js -o public/bundle.js -v -p mapstraction --debug
   ```

7. A compound VSCode launch task to watch for changes automatically and rebundle things.
   Here's an example task that runs the watchify npm script which can be used in a VSCode launch:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build-full",
      "command": "npm run build:dev",
      "type": "shell"
    }
  ]
}
```

### Enter Webpack

Now we've got a nice module bundler pipeline going and a dev workflow that is not intrusive. After a day of work to get the project to this point, I certainly felt like I was #winning. I was not going to get three Webpack strikes.

##### Injecting Bundles, The Final Frontier:

The last piece of the puzzle is to inject our hashed (cache busting) bundles into a pug file. In the Gulp world, I used gulp-inject which worked great. This is the hangup with Browersify, it fits into a build pipeline while Webpack can be the build pipeline. This was the last piece I needed. I could probably write a plugin to do this but it would feel weird. Plugins in Browersify go off "Transforms". The transform function fires for every file in the current package and returns a transform stream that performs the conversion. Not ideal. There are a multitude of ways to handle this problem, but they all rely on adding more pieces to the puzzle instead of using existing piece. I want to keep the puzzle small.
At this point, it's either change the way our pug file works, use Gulp, or write a hacky solution. Option 1 is not going to work, I don't want to impact other devs and the whole reason we're going through this exercise is to make things better and move away from Gulp.
Here's an example of the Gulp task I was using to build the bundle:

```javascript
Upgrading an Angular1x app to ES2015 Syntax

var babelify = require('babelify')
var browserify = require('browserify')
var vinylSourceStream = require('vinyl-source-stream')
var vinylBuffer = require('vinyl-buffer')

/* Compile all script files into one output minified JS file. */
gulp.task('bundlify', function () {
 var sources = browserify({
   entries: [
     'public/app/app.js'
   ],
   debug: true // Build source maps
 })
 .transform(babelify.configure({
   presets: ['@babel/preset-env']
 }))
 .transform(bulkify)

 return sources.bundle()
 .pipe(vinylSourceStream('main.min.js'))
 .pipe(vinylBuffer())
 .pipe($.sourcemaps.init({
   loadMaps: true // Load the sourcemaps browserify already generated
 }))
 .pipe($.ngAnnotate())
 .pipe($.uglify())
 .pipe($.sourcemaps.write('./', {
   includeContent: true
 }))
 .pipe(gulp.dest('./dist'))
})
}

```

We've come so far, won many battles => moving the modules to ES modules, shimming globals, removing Bower from the process, getting our app bundled. We're going to need Webpack to win the war though and finally excise Gulp from the project.

[Webpack](https://webpack.js.org/concepts/){:target="\_blank"} is a vastly configurable static module bundler.  
Reasons for moving to Webpack:

- I need to inject sources to align with the current build process which uses Gulp. I want to remove Gulp from the process
- I want to bundle styles, I know I could probably do this with Browersify but I didn't get to that point yet.
- Configuration based: even though configuring Webpack is more complex than Browersify, I thought the configuration nature would be easier for future developers to comprehend and extend.
- It's popular, this one hurts to say as I really bonded with Browersify and their ethos. It fits my style, 100%. However, as an enterprise application, the well-known option has it's benefits.

##### Webpack Crash Course:

_Entry_: which module Webpack should use to begin building out its internal dependency graph. Basically where things start, for us it's app.js.

_Output_: Where bundles go

_Loaders_: Processes types of files. Two properties:

- test: which files types should be transformed (usually regexes are used /\.js\$/)
- use: what loader (processor) to use on those files

_Plugins_: Used for more functionality than transforms (minification, asset optimization, generate an html file, etc).

_Mode_: Development, Production, None. Built-in optimizations happen for production mode.

##### Webpack Conversion

1.  Replace bulk-require and bulkify with Webpack's [require.context](https://webpack.js.org/guides/dependency-management/#require-context){:target="\_blank"}.
    The bulk-require solution felt like a hack while Webpack's require.context is essentially the same functionality natively supported:

    After:

    ```javascript
    angular.module("blocks.logger", []); // create the module
    function importAll(r) {
      _.forEach(r.keys(), r);
    }
    importAll(
      require.context(
        "./",
        true,
        /^(?!.*\.module\.js$)^(?!.*\.spec\.js$).*\.js$/
      )
    );
    export default angular.module("blocks.logger"); // export our module
    ```

2.  Get a working Webpack config going to bundle Javascript. Use Webpack's [ProvidePlugin](https://webpack.js.org/plugins/provide-plugin/){:target="\_blank"} to expose globals.

    ```javascript
    const webpack = require("webpack");
    const path = require("path");

    module.exports = {
      mode: "none",
      entry: {
        app: path.join(__dirname, "/public/app/app.js")
      },
      output: {
        path: path.join(__dirname, "/public/"),
        filename: "[name].js"
      },
      devtool: "eval-source-map",
      module: {
        rules: [
          {
            test: /\.js$/,
            use: {
              loader: "babel-loader",
              options: {
                presets: ["@babel/preset-env"]
              }
            },
            exclude: /node_modules/
          }
        ]
      },
      // Use ProvidePlugin to expose jQuery to the window object, replaces /browersify-shim:
      plugins: [
        new webpack.ProvidePlugin({
          "window.$": "jquery",
          "window.jQuery": "jquery",
          $: "jquery"
        })
      ]
    };
    ```

3.  Include styles. This project uses sass. In app.js we're going to import our sass files and use the sass-loader (npm install sass-loader -D)

    app.js

    ```javascript
    import "../assets/scss/styles.scss";
    ```

    webpack.config.js

    ```javascript
    {
        test: /\.s[ac]ss$/i,
        use: [
          // Creates `style` nodes from JS strings
          'style-loader',
          // Translates CSS into CommonJS
          'css-loader',
          // Compiles Sass to CSS
          'sass-loader'
        ]
    }
    ```

    [autoprefixer](https://github.com/postcss/autoprefixer) is something else to look at, it parses css and adds vendor rules.

4.  Development and Production Webpack configurations - Webpack Merge
    `npm install webpack-merge`
    webpack.dev.js -> replaces watchify, watch: true will watch the bundle files and rebuild. You can use the --silent option to suppress the output.

    webpack.dev.js

    ```javascript
    const merge = require('webpack-merge')
    const common = require('./webpack.config.js')
    const path = require('path')

    module.exports = merge(common, {
      mode: 'development',
      devtool: 'inline-source-map',
      output: {
        path: path.join(__dirname, '/public/'),
        filename: '[name].js'
      },
      watch: true
      plugins: []
    })
    ```

    For Production:

    - mode: set this to production
    - Minification: terser-webpack-plugin and optimize-css-assets-webpack-plugin
    - Copy Files to Dist Directory: copy-webpack-plugin
    - Clean Dist Directory: clean-webpack-plugin
    - Cache-Busting: Use hash in output
    - Extract CSS into separate bundle to decrease bundles sizes: mini-css-extract-plugin

    webpack.prod.js

    ```javascript
    const webpack = require("webpack");
    const merge = require("webpack-merge");
    const common = require("./webpack.config.js");
    const path = require("path");

    const TerserJSPlugin = require("terser-webpack-plugin");
    const OptimizeCSSAssetsPlugin = require("optimize-css-assets-webpack-plugin");

    const MiniCssExtractPlugin = require("mini-css-extract-plugin");
    const CopyPlugin = require("copy-webpack-plugin");
    const { CleanWebpackPlugin } = require("clean-webpack-plugin");

    module.exports = merge(common, {
      mode: "production",
      devtool: false,
      output: {
        path: path.resolve(process.cwd(), "dist"),
        publicPath: "",
        filename: "[name].[hash].js"
      },
      module: {
        rules: [
          {
            test: /\.s[ac]ss$/i,
            use: [
              MiniCssExtractPlugin.loader,
              "css-loader",
              "postcss-loader",
              "sass-loader"
            ]
          }
        ]
      },
      optimization: {
        minimizer: [new TerserJSPlugin(), new OptimizeCSSAssetsPlugin()]
      },
      plugins: [
        new CleanWebpackPlugin(),
        new CopyPlugin([
          { from: "app/**/*.html", context: "public" } // TODO: need to figure out template cache with webpack
        ]),
        new MiniCssExtractPlugin({
          filename: "[name].[hash].css",
          chunkFilename: "[id].css"
        })
      ]
    });
    ```

5.  Injecting bundles  
    We're finally to the point we were with Browersify plus we've got our sass files imported now. Injecting the hashed bundles into a pug file.  
    This is where I got stuck for a little bit. The html-webpack-plugin is okay but it mainly focuses on generating a new index file. There are [pug plugins](https://github.com/negibouze/html-webpack-pug-plugin){:target="\_blank"} but none of them are as seamless as gulp-inject. Basically in the pug file we have marker comments like //- inject:js //- endinject. And the files are injected between those comments.
    Webpack has a dynamic plugin architecture, so I ended up writing my [own naive plugin](https://github.com/Juxly/pug-gulp-inject-webpack-plugin){:target="\_blank"} to replace the gulp-inject functionality. It's basic and doesn't support SplitChunks at the moment but it gets the job done.

    ```javascript
    const InjectPlugin = require("pug-gulp-inject-webpack-plugin");

    new InjectPlugin({
      template: "views/includes/head.jade",
      output: path.join(process.cwd(), "views/includes/head.jade")
    });
    ```

### Bundle Size Optimization: Bonus Round

Two useful tools for tracking down bundle size issues:

[discify](https://github.com/131/discify){:target="\_blank"}: Browersify plugin that generates graph and stats of your bundle

[source-map-explorer](https://github.com/paulirish/source-map-explorer){:target="\_blank"}: Analyze and debug JavaScript (or Sass or LESS) code bloat through source maps.

Slimming down moment and moment-timezone:

I'm able to get by only shipping the en-us locale with moment which saves some space.

```javascript
new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/); // ignore all locales by default, only ship with en-us
```

moment-timezone ships with a ton of data, to slim it down, you can change the import to only bring in a ten year span of data:

```javascript
import momentTz from "moment-timezone/builds/moment-timezone-with-data-2012-2022";
```

Webpack Chunk Splitting: More on this in the future but I'm currently using two entry points to generate two separate bundles. It's the basic form of bundle splitting which doesn't really allow deduplication but that's okay in my case for now.

##### Conclusion

The journey from Gulp to Webpack for this AngularJS application is mostly complete. It took getting Browersify involved to finally be able to make the transition to Webpack for a 2014 AngularJS app. There are still more hills to climb, getting AngularJS's template cache working and better bundle splitting but this is a good start. Now that we can write frontend javascript with ES-whatever, the sky is the limit. Maybe we start transitioning to Svelte? :D

If you read this far, give me a shoutout on [Twitter](https://twitter.com/james_ingold) / send any questions or comments to [yo[@]jamesingold.com](mailto:yo@jamesingold.com)

##### Further Reading / References:

[Javascript Modules - A Beginner's Guide](https://www.freecodecamp.org/news/javascript-modules-a-beginner-s-guide-783f7d7a5fcc/){:target="\_blank"}

[ng-book: The Complete Book on AngularJS](https://amzn.to/3dI1c5w){:target="\_blank"}

[Browersify for Webpack Users](https://gist.github.com/substack/68f8d502be42d5cd4942){:target="\_blank"}

[Browersify Handbook](https://github.com/browserify/browserify-handbook){:target="\_blank"}

[Reduce moment-timezone data size with Webpack](https://github.com/gilmoreorless/moment-timezone-data-webpack-plugin){:target="\_blank"}

[Github Issue Megathread on Moment Locales / General Size Issues](https://github.com/moment/moment/issues/2517){:target="\_blank"}

[Code-Splitting in Webpack](https://webpack.js.org/guides/code-splitting/){:target="\_blank"}

[Compound Builds in VSCode](https://medium.com/@auchenberg/introducing-simultaneous-nirvana-javascript-debugging-for-node-js-and-chrome-in-vs-code-d898a4011ab1#.bgkqbh54p){:target="\_blank"}
