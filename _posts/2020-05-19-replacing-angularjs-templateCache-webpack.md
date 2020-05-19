---
title: Replacing AngularJS $templateCache with Webpack
description: Replacing AngularJS $templateCache with Webpack
author: James Ingold
published: true
---

| AngularJS Webpacking Posts                                                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Part One - Megathread to Convert Gulp/Bower Build to Webpack in AngularJS](https://dev.to/jamesingold/gulp-bower-browserify-webpack-an-angularjs-journey-4eic) |
| [Part Two - You are here now](#)                                                                                                                                |

In part two of transitioning a Gulp/Bower built AngularJS app to Webpack, we'll be picking up where we left off: replacing AngularJS's \$templateCache with Webpack.

We left off with a production Webpack configuration that contained a plugins section somewhat like this:

```javascript
// webpack.prod.js
plugins: [
  new CleanWebpackPlugin(),
  new CopyPlugin([
    { from: "app/**/*.html", context: "public" }, // TODO: need to figure out template cache with webpack
    { from: "favicon.ico", to: "favicon.ico", context: "public" }
  ]),
  new MiniCssExtractPlugin({
    filename: "[name].[hash].css",
    chunkFilename: "[id].css"
  }),
  new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/) // ignore all locales by default, only ship with en-us
];
```

We're going to figure out that TODO item.

### What is the \$templateCache in AngularJS?

It's a simple key-value store returned from the $cacheFactory. The $templateCache is a cache of your templates. The key is the url to the template and the value is the HTML content.

#### How it works:

When you request a template, AngularJS will check the \$templateCache service first. If it's found, great, you've saved a request to the server. If it's not then the template is retrieved and it gets stored in the cache. From there, it can quickly be retrieved on subsequent calls.
Or calling the cache directly:

```javascript
$templateCache.get("mytemplate.html");
```

What we were doing on builds was pre-populating the \$templateCache, so that the very first call is a cache hit and not a cache miss.
Here's the gulp build task that did the building using gulp-angularTemplateCache:

```javascript
gulp.task("templatecache", function() {
  return gulp
    .src("app/**/*.html", { cwd: path + "/public/" })
    .pipe(
      $.angularTemplatecache("templates.js", {
        module: "templateCache",
        standalone: true,
        root: "/app/"
      })
    )
    .pipe(gulp.dest("dist"));
});
```

### Webpack HTML Templates - Enter html-loader

In part one, we were moving all of the HTML files to the dist/ folder. This was somewhat by design, as I wanted to modify the least amount of code as possible to make the move to Webpack a seamless transition for developers on the team. Now that Webpack is settled in as the build process and we're not getting tons of bug reports, it's time to tie up the loose ends. We want to get those HTML templates/fragments into our bundles.

There are two options: continuing to use AngularJS's \$templateCache: build a templates.js file and import it into app.js or use Webpack.

I initially set out to use the $templateCache because that's what the app was doing before Webpack. But after reading more about [html-loader](https://webpack.js.org/loaders/html-loader/) in Webpack, I decided to get rid of the $templateCache all together.

html-loader: exports HTML as a string. Any time we require an html file, it will be returned from our bundle. Super speedy like. This functionality completely replaces the \$templateCache and there is no reason to continue to use this native AngularJS service when we're going to move away from the framework in the future.

### The Process

1. require() all templates, change templateUrls to templates. Since the html-loader will be returning the HTML string, we need to use the template property instead of templateUrl which expects a file path. All of our template urls were pathed to the app folder, we can just use the relative file when requiring.

   ```javascript
   templateUrl: "/app/analytics/analytics.html";
   ```

   To:

   ```javascript
   template: require("./analytics.html");
   ```

2. Transform ng-includes into components. ng-includes promotes using HTML fragments which are usually accompanied by an ng-controller= "controller as vm". This was okay five years ago, but now it's better to use AngularJS components (if you still have to use AngularJS :D)

   index.html (or index.pug in my case)

   ```html
   <div
     id="Header"
     class="fixed-top border-bottom"
     ng-include="'/app/header/myheader.html'"
   ></div>
   ```

   header.component.js

   ```javascript
   const module = angular.module("app.header");

   module.component("myHeader", {
     bindings: {},
     controllerAs: "vm",
     controller: "headerController",
     template: require("./myheader.html")
   });
   ```

   index file, componentized

   ```html
   <my-header />
   ```

3. Make sure you're using _valid_ html, as html-loader will error if there are issues. I had one problem where single quotes were used in an attribute like class=\'red\' and html-loader did not like that.

4. Clean up app.js - I had some previous TODO comments to remove and I could delete the old templates.js file

   ```javascript
       -// TODO: template cache?
       -// require('./templates')
   ```

5. Remove copy plugin line from webpack.prod.js:

   ```diff
   -  { from: 'app/**/*.html', context: 'public' }, // TODO: need to figure out template cache with webpack
   ```

### Wrapping Up

We've removed a uniquely AngularJS \$templateCache service with Webpack and moved some HTML fragments to components. All of this will be of use as we slowly transition away from AngularJS. Also, in Webpack's production mode, the HTML will be minified which is great. We don't need to modify our Webpack configuration further for deployments. We've added another notch in our Webpack belt as our templates are now bundled. The next steps are better bundle splitting and potentially using the file-loader plugin for the images. Stay tuned!
