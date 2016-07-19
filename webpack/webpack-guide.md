# WEBPACK GUIDE

This is a quick, on demand and yet unedited/untested guide how to set up [webpack](https://webpack.github.io) build system (in the moment you drop [gulp](http://gulpjs.com)).

Firstly it uses simple SCSS, plain JavaScript ES5. Then drops in loaders/plugins for CSS and Javascript ES6/ES2015 (PostCSS + plugins, Babel, ESLint a.o.). Finally it adds React.js in the mix.

webpack 1.13.x-1.14.x assumed.

Put together by @kroko for the new colleagues that see webpack for the first time.  
While doing first edits I realised that also notes should be made for some basic npm stuff as the reality is - there are people who haven't used any building tools or even npm before (which isn't bad thing if you haven't coded JavaScript and/or do backend (PHP/ROR/...) stuff only). So this assumes absolute entry level knowledge in terms of packing code.

* clone it
* read it
* learn it
* create new project in our server and test it step by step
* there are things that are left out (especially server proxy config for `webpack-dev-server`, so ask if you don't see connection from some A to B)
* watch out for errors (lot of stuff here is untested, as I don't have such not-real-world (barebone) code anywhere) as well as things that simply do not work anymore, because there is better way to do it (webpack gets updates, you know...)
* add, commit and push fixes/changes/additions to this repo so that we can make this the ultimate webpack guide.

## Result

### Final `webpack.config.js`

Looks something like this

```javascript
'use strict';

const path = require('path');
const webpack = require('webpack');
const ExtractTextPlugin = require('extract-text-webpack-plugin');

const production = process.env.NODE_ENV === 'production';
const testing = process.env.NODE_ENV === 'testing';

console.log(production ? 'This is production config' : 'This is dev config');

const webpackHtaccess = require('manage-htaccess');

webpackHtaccess(
  [
    {
      tag: 'DUMMY',
      enabled: false
    },
    {
      tag: 'WARPDEV',
      enabled: !production,
      attributes: {
        port: 3334
      }
    }
  ],
  path.join(__dirname, 'public/.htaccess')
);

let config = {
  // create source maps only on development or staging, not production
  devtool: (production) ? null : 'inline-source-map',
  target: testing ? 'node' : 'web',

  context: __dirname,
  entry: {
    preflight: './src/preflight.js',
    site: './src/site.js'
  },
  output: {
    path: './public/assets', // path.join(__dirname, 'public/public')
    filename: '[name].js',
    publicPath: production ? '//ma.server.tld/public/assets/' : 'http://ma.server.tld/public/assets/'
  },
  resolve: {
    modulesDirectories: [
      'src',
      'node_modules',
      'bower_components'
    ],
    root: path.resolve('./src/')
  }
};

config.module = {
  // noParse: [
  //   /preflight\.js$/
  //   // path.join(__dirname, 'src/preflight.js')
  // ],
  preLoaders: [
    {
      test: /\.js$/,
      exclude: [/node_modules/, /preflight\.js$/],
      loader: 'eslint-loader'
    }
  ],
  loaders: [
    {
      test: /\.js$/,
      exclude: [/node_modules/, /preflight\.js$/],
      loader: production ? 'babel-loader' : 'react-hot-loader!babel-loader'
    },
    {
      test: /\.(scss)$/,
      loader: production
      ? ExtractTextPlugin.extract('css-loader!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap') // postcss-loader!resolve-url-loader!
      : 'style-loader!css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap'
    },
    {
      test: /\.(css)$/,
      loader: production
        ? ExtractTextPlugin.extract('css-loader?sourceMap!postcss-loader')
        : 'style-loader!css-loader?sourceMap!postcss-loader'
    },
    {
      test: /\.(png|jpg|gif)$/,
      loader: production
        ? 'url-loader?limit=10000!image-webpack-loader'
        : 'url-loader?limit=10000'
    },
    {
      test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader'
    },
    {
      test: /\.woff2(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=application/font-woff'
    },
    {
      test: /\.woff(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=application/font-woff'
    },
    {
      test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=application/x-font-ttf'
    },
    {
      test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=image/svg+xml'
    }
  ]
};

// ############################################################################
// Plugins
// ############################################################################

config.plugins = [];

// define enviromental variable into script files
config.plugins.push(new webpack.DefinePlugin({
  __CLIENT__: true,
  __SERVER__: false,
  __DEVELOPMENT__: !production,
  __DEVTOOLS__: !production,
  'process.env': {
    NODE_ENV: JSON.stringify(process.env.NODE_ENV || 'development')  }
}));

if (production) {
  config.plugins.push(new ExtractTextPlugin('[name].css', {
    allChunks: true
  }));
}

if (production) {
  config.plugins.push(new webpack.optimize.UglifyJsPlugin({
    sourceMap: false,
    test: /\.js($|\?)/i,
    compressor: {
      drop_console: true, // do this always
      // sequences: true,
      // dead_code: true,
      // drop_debugger: true,
      // unused: true,
      // join_vars: true,
      warnings: false
    }
  }));
}

if (production) {
  config.plugins.push(new webpack.optimize.DedupePlugin());
}

// ############################################################################
// 3rd party loader and plugin configuration
// ############################################################################

config.eslint = {
  quite: !production,
  failOnWarning: false,
  failOnError: production
};

// https://github.com/postcss/postcss-loader
config.postcss = function () {
  return [
    require('autoprefixer')({
      browsers: ['> 0.0001%'],
      cascade: true,
      remove: true
    }),
    require('css-mqpacker')(),
    require('cssnano')({
      discardComments: {
        removeAll: true
      },
      zindex: false
    })
    /*
    ["zindex", "normalizeUrl", "discardUnused", "mergeIdents", "reduceIdents"]
    */
  ];
};

module.exports = config;
```

### Final `package.json`

```json
{
  ...
  "scripts": {
    "build:dev": "npm run build:clean && webpack-dev-server --host=my.host.tld --port=3333 --history-api-fallback -d --inline --hot --content-base ./public/assets",
    "build:prod": "npm run build:clean && NODE_ENV=production webpack --progress",
    "build:clean": "rm -rf ./public/assets/*",
    "build": "npm run build:prod",
    ...
  }
  ...
}
```

---
# How to get to the result?
===

## PREFLIGHT

### SET UP BASIC FILES & DIR STRUCTURE

Crate master directory and set files tree up like this (`tree -a .`). Leave all files empty, we will fill them up step by step.

```
master-directory
├── package.json
├── public
│   ├── .htaccess
│   └── index.html
├── src
│   ├── global.scss
│   ├── preflight.js
│   └── site.js
└── webpack.config.js
```

*public/index.html*

```html
<!DOCTYPE html>
<html class="noscript">
<head>
  <meta charset="utf-8">
  <title>My Title</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <script src="./assets/preflight.js"></script>
  <link rel="stylesheet" type="text/css" href="./assets/site.css">
</head>
<body>
  <div class="app"></div>
  <script>
    var dataReact = {};
  </script>
  <script async src="./assets/site.js"></script>
</body>
</html>
```

Leave `webpack.config.js`, `src/site.js` and `src/global.scss` empty for now. Either leave `package.json` out and generate it in the next step (`npm init`) or put our template `package.json` in place / [manually fill in](https://docs.npmjs.com/files/package.json) bare minimum yourself.

### NPM INIT

`cd` in master directory and initialise npm in it. Not needed if you have placed template or DIY `package.json` already there.

```sh
npm init
```

## Vanilla JavaScript and SCSS/CSS setup

*Note for [absolute beginners](https://www.youtube.com/watch?v=r8NZa9wYZ_U). All `npm` as well as `webpack` commands are executed while being `cd`-ed in this projects 'master directory'. You can do it while being somewhere else via `npm --prefix ${DIRNAME} install ${DIRNAME}` though. RTFM@NPM / ask.*

### Set up webpack so that we can simply build js file

Install webpack and save to dev dependencies

```sh
npm install webpack --save-dev
```

Fill in webpack configuration file. Please always use ES6 in webpack config files.

_webpack.config.js_

```javascript
'use strict';
const path = require('path');
let config = {
 
  context: __dirname,
  entry: {
    site: './src/site.js' // path.join(__dirname, 'src/site.js') if context not set
  },
  output: {
    path: './public/assets', // path.join(__dirname, 'public/assets') if context not set
    filename: '[name].js'       
  },
  resolve: {
    modulesDirectories: [
      'src',
      'node_modules',
      'bower_components'
    ],
    root: path.resolve('./src/')
  }
}
module.exports = config;
```

Set up hello world javascript that selects `app` div in our html and puts some text in it. Note that we are using ES5 here.

_site.js_

```javascript
'use strict';
var div = document.querySelector('.app');
div.innerHTML = 'Hello JS';
console.log('Hello JS!');
```

Run webpack

```sh
webpack --progress
```

Notice, that entry point __key names__ dictate what will be the outputted __filename__ in `./public/assets`. That is, you can change key name and real file name to whatever, i.e.,

```javascript
entry: {
  myBundle: './src/whatever.js'
},
```

and you will get `myBundle.js` in `./public/assets` (later you will see that CSS that is imported through JavaScript also will be named as the entry point name, i.e., `./public/assets/myBundle.css`).

You can change this behaviour if output `filename: '[name].js'` is set to `filename: 'someConstantName.js'`.  
But don't do that, let your entry point key name define the output name.  
Think of what would happen if you had multiple entry points (just like in a real world scenario). How would you manage filenames then if output file would not somehow depend on entry point, but would be always constant?  
And yeah, we will not discuss filename based versioning stuff in this tut.


### Build multiple js files

Our template has `preflight.js` in the head which is not present after building in previous step (and we get `404`). So let us add new endpoint for preflight.

_preflight.js_

```javascript
// change noscript to script in html tag
document.documentElement.className = document.documentElement.className.replace(/\bnoscript\b/, 'script');
```

_webpack.config.js_

```javascript
'use strict';
const path = require('path');
let config = {
 
  context: __dirname,
  entry: {
    site: './src/site.js',
    preflight: './src/preflight.js'
  },
  output: {
    path: './public/assets',
    filename: '[name].js'       
  },
  resolve: {
    modulesDirectories: [
      'src',
      'node_modules',
      'bower_components'
    ],
    root: path.resolve('./src/')
  }
}
module.exports = config;
```

Run webpack

```sh
webpack --progress
```

Inspect. The file is in `public/assets` directory and it does its job.


### Webpack minimise JavaScript

For JavaScript minimisation we can use webpack built in plugin which actually uses [UglifyJS2](https://github.com/mishoo/UglifyJS2) under the hood.

Plugin documentation  
[webpack.optimize.UglifyJsPlugin](https://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin)

_webpack.config.js_

```javascript
'use strict';
const path = require('path');

const production = process.env.NODE_ENV === 'production';

let config = {
 
  context: __dirname,
  entry: {
    site: './src/site.js',
    preflight: './src/preflight.js'
  },
  output: {
    path: './public/assets',
    filename: '[name].js'       
  },
  resolve: {
    modulesDirectories: [
      'src',
      'node_modules',
      'bower_components'
    ],
    root: path.resolve('./src/')
  }
}

const webpack = require('webpack');
config.plugins = []; // add new key 'plugins' to config object, array
if (production) {
  config.plugins.push(new webpack.optimize.UglifyJsPlugin({
    compressor: {
      warnings: false
    }
  }));
}

module.exports = config;
```

Run webpack, specify `NODE_ENV` value

```sh
NODE_ENV=production webpack --progress
```

Inspect how `assets/site.js` changes based on whether `NODE_ENV` is set to `production`.

### node-sass

For to compile SCSS to CSS we will be using [Node-sass](https://github.com/sass/node-sass). It _is a library that provides binding for Node.js to LibSass, the C version of the popular stylesheet preprocessor, Sass_.  

```sh
npm install node-sass --save-dev
```

### Webpack CSS loaders so that CSS/SCSS can be required in js

So we need a bunch of webpack loaders for this to work and to get that `site.css` working that is ref'ed in the `<head>`.

Loaders and their documentation  
[https://github.com/webpack/style-loader](https://github.com/webpack/style-loader)  
[https://github.com/webpack/css-loader](https://github.com/webpack/css-loader)  
[https://github.com/jtangelder/sass-loader](https://github.com/jtangelder/sass-loader)  

Plugin and its documentation  
[https://github.com/webpack/extract-text-webpack-plugin](https://github.com/webpack/extract-text-webpack-plugin)  

[extract-text-webpack-plugin](https://github.com/webpack/extract-text-webpack-plugin) moves every `require('style.css')` within JavaScript that is spilled out in chunks into a separate css output file. So your styles are not inlined into the JavaScript (which would be hind of default webpack way without this plugin), but separate in a css bundle file `entryPointKeyName.css`.

```sh
npm install style-loader --save-dev
npm install css-loader --save-dev
npm install sass-loader --save-dev
npm install extract-text-webpack-plugin --save-dev
```

_webpack.config.js_

```javascript
'use strict';
const path = require('path');
const webpack = require('webpack');
const ExtractTextPlugin = require('extract-text-webpack-plugin');

const production = process.env.NODE_ENV === 'production';

// ----------------
// BASE CONFIG

let config = {
 
  context: __dirname,
  entry: {
    site: './src/site.js',
    preflight: './src/preflight.js'
  },
  output: {
    path: './public/assets',
    filename: '[name].js'       
  },
  resolve: {
    modulesDirectories: [
      'src',
      'node_modules',
      'bower_components'
    ],
    root: path.resolve('./src/')
  }
}

// ----------------
// MODULES

config.module = {
  loaders: [
    {
      test: /\.(scss)$/,
      loader: ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!sass-loader?sourceMap')
    }
  ]
}

// ----------------
// PLUGINS

config.plugins = [];

config.plugins.push(new ExtractTextPlugin('[name].css', {
  allChunks: true
}));

if (production) {
  config.plugins.push(new webpack.optimize.UglifyJsPlugin({
    compressor: {
      warnings: false
    }
  }));
}

module.exports = config;
```

_site.js_

```javascript
'use strict';

require('./global.scss');

var div = document.querySelector('.app');
div.innerHTML = 'Hello JS';
console.log('Hello JS!');
```

_global.scss_

```scss
.app {
  background-color: red;
}
```

Run webpack and inspect `public/assets/site.css`

```sh
webpack --progress
```

SCSS is compiled and spit out in a file under `public/assets` named the same as the entry point key of the JavaScript from which SCSS was included in the first place.

### Note about loader names

Whenever you use loaders

```javascript
('style-loader', 'css-loader?sourceMap!sass-loader?sourceMap')
```

you can omit `-loader` part

```javascript
('style', 'css?sourceMap!sass?sourceMap')
```

But IMHO it's better to be explicit as it helps finding, batch replacing things.

### PostCSS plugins

One does not simply... don't use PostCSS. [Use plugins!](https://cdn.meme.am/instances/500x/68322636.jpg)

Loader  
[postcss-loader](https://github.com/postcss/postcss-loader)  

Basic plugins and their documentation  
[autoprefixer](https://github.com/postcss/autoprefixer)  
[node-css-mqpacker](https://github.com/hail2u/node-css-mqpacker)  
[cssnano](https://github.com/ben-eb/cssnano)

```sh
npm install postcss-loader --save-dev
npm install css-mqpacker --save-dev
npm install autoprefixer --save-dev
npm install cssnano --save-dev
```

First add PostCSS plugin configuration. Do some sick backwards browser support for a test.  
Then also add `postcss-loader` in the loaders pipe.

_webpack.config.js_

```javascript
...
      loader: ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!postcss-loader!sass-loader?sourceMap')
...

config.postcss = function () {
  let postPluginConf = [];
  postPluginConf.push(
    require('autoprefixer')({
      browsers: ['> 0.0001%'],
      cascade: true,
      remove: true
    })
  );
  postPluginConf.push(
    require('css-mqpacker')()
  );
  // we minimize CSS always as we have source maps, but for this example let us do conditional
  if (production) {
    postPluginConf.push(
      require('cssnano')({discardComments: {removeAll: true}, zindex: false})
    );
  }
  return postPluginConf;
};
...

```

_global.scss_

```scss
.app {
  background-color: red;
  display: flex;
  transform: translateY(50px);
}
```

Run webpack and inspect `public/assets/site.css`

```sh
webpack --progress
```

### normalize.css

Always use [Normalize.css](https://necolas.github.io/normalize.css/). Although our *cut the mustard* script does fallback page for anything below IE11 this fallback has to look O.K., so use Normalize.css that supports IE8+ (which is 4.x as of now.)

```sh
npm install normalize.css --save-dev
```

We add it as a module, so prefix it `~`. So now you know what `resolve: { modulesDirectories: [] }` in webpack stands for. Search paths!

Add to our SCSS

_site.scss_

```scss
@import "~normalize.css";
.app {
  background-color: red;
  display: flex;
  transform: translateY(50px);
}
```

Run webpack and inspect `public/assets/site.css`

```sh
webpack --progress
```

### Webpack CSS source maps

As you can see we pass `?sourceMap` to our loaders (currently we do not have any loaders for JavaScript). But where are the source maps? Use webpack [devtool](https://webpack.github.io/docs/configuration.html#devtool).

First try

_webpack.config.js_

```javascript
config = {
  devtool: 'inline-source-map',
  ...
}
```

Run webpack and inspect `public/assets/site.js` and `public/assets/site.css`

```sh
webpack --progress
```

Now try

```javascript
config = {
  devtool: 'source-map',
  ...
}
```

Run webpack and inspect `public/assets/` directory.

Finally set for now to always generate source maps, as we are testing things here

_webpack.config.js_

```javascript
config = {
  devtool: production ? 'source-map' : 'inline-source-map',
  ...
}
```

In real world I tend to use

```javascript
devtool: (production) ? null : 'inline-source-map',
```

which would enable source maps both for development and staging.


### Webpack file-loader & url-loader & resolve-url-loader

Now let us add some images to source. Make `my-small-image.jpg` few KB and `my-large-image.jpg` above  1 MB.

`tree -a -I 'node_modules' .`

```
├── package.json
├── public
│   ├── assets
│   │   ├── preflight.js
│   │   ├── site.css
│   │   └── site.js
│   ├── .htaccess
│   └── index.html
├── src
│   ├── global.scss
│   ├── images
│   │   ├── my-large-image.jpg
│   │   └── my-small-image.jpg
│   ├── preflight.js
│   └── site.js
└── webpack.config.js
```

Loaders  
[file-loader](https://github.com/webpack/file-loader)  
[url-loader](https://github.com/webpack/url-loader)  
[resolve-url-loader](https://github.com/bholloway/resolve-url-loader)  

`url-loader` works like the file loader, but can return a Data Url if the file is smaller than a limit.  
`resolve-url-loader` is needed because we set file path in `url()` relative to the SCSS file we are working in, however webpack tries to resolve from entry point.

```sh
npm i -D file-loader
npm i -D url-loader
npm install resolve-url-loader --save-dev
```

Add image to our SCSS

_site.scss_

```scss
@import "~normalize.css";
body {
  background-image: url('./images/my-large-image.jpg');
}
.app {
  background-color: red;
  display: flex;
  transform: translateY(50px);
  height: 100px;
  background-image: url('./images/my-small-image.jpg');
}
```

Add loader in the pipe and loader rule to handle this file type (don't be suprised about not including svg, later on that)

_webpack.config.js_

```javascript

...
      loader: ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap')
...
    {
      test: /\.(png|jpg|jpeg|gif)$/,
      loader: 'url-loader?limit=10000'
    }
...
```

Run webpack and inspect `public/assets/` directory and `public/assets/site.css`

```sh
rm -rf public/assets/** && webpack --progress
```

Notice how larger image is outputted as file while smaller image is inlined `url(data:image/jpeg;base64,...);` in CSS. Just as we want it.

### Webpack image-webpack-loader

But we can do better. For those images that are outputted as files, let us minify them.  
_Minify PNG, JP(E)G, GIF and SVG images with [imagemin](https://github.com/imagemin/imagemin)._

Loader  
[image-webpack-loader](https://github.com/tcoopman/image-webpack-loader)  

```sh
npm i -D image-webpack-loader
```
_webpack.config.js_

```javascript
...
    {
      test: /\.(png|jpg|jpeg|gif)$/,
      loader: 'url-loader?limit=10000!image-webpack-loader'
    }
...
```

Run webpack and inspect `public/assets/` directory, how outputted image size differs from source.

```sh
rm -rf public/assets/** && webpack --progress
```

This process is expensive. In development we do not care about filesizes as we live within gigabit ethernet, so set it to 

```javascript
...
    {
      test: /\.(png|jpg|jpeg|gif)$/,
      loader: production
        ? 'url-loader?limit=10000!image-webpack-loader'
        : 'url-loader?limit=10000'
    }
...
```

-----
###### REVISED TILL HERE


### Webpack webpack-dev-server

[https://github.com/webpack/webpack-dev-server](https://github.com/webpack/webpack-dev-server)

```sh
npm i -D webpack-dev-server
```

### Webpack style-loader

We have alreaddy installed this. Adds CSS to the DOM by injecting a `<style>` tag. Use with webpack-dev-server.
[https://github.com/webpack/style-loader](https://github.com/webpack/style-loader)

```sh
npm i -D style-loader
```

### manage-htaccess

[https://github.com/WARP-LAB/manage-htaccess](https://github.com/WARP-LAB/manage-htaccess)

```sh
npm install manage-htaccess --save-dev
```

### Font building

todo

### Font copying

_webpack.config.js_

```javascript
loaders: [
    // Troubleshooting Webpack "OTS parsing error" loading fonts.
    // Note: NOT FIXED!
    // Ref: http://stackoverflow.com/a/34133809/1378261
    // Ref: http://stackoverflow.com/a/30763977/1378261
    {
      test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader'
    },
    {
      test: /\.woff2(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=application/font-woff'
    },
    {
      test: /\.woff(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=application/font-woff'
    },
    {
      test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=application/x-font-ttf'
    },
    {
      test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
      loader: 'file-loader?mimetype=image/svg+xml'
    }
]
```



### Webpack GLSL loader

[phaser-glsl-loader](phaser-glsl-loader)

_webpack.config.js_

```javascript
  loaders: [
    {
      test: /\.(frag|vert)/,
      loader: 'phaser-glsl-loader'
    },
    ...
```
_site.js_

```javascript
  shaderProg = new THREE.ShaderMaterial({
    uniforms,
    vertexShader: require('./shader.vert').join('\n'),
    fragmentShader: require('./shader.frag').join('\n')
  });
```

### Webpack expose loader

[expose loader](https://github.com/webpack/expose-loader)  
When wanting to put things in global namespace

```
npm install expose-loader --save-dev
```

### Webpack 

[script-loader](https://github.com/webpack/script-loader)  

```
npm install script-loader --save-dev
```

### Webpack plugins, other

[webpack.DefinePlugin](https://webpack.github.io/docs/list-of-plugins.html#defineplugin)  
Define free variables. Useful for having development builds with debug logging or adding global constants. The values will be inlined into the code which allows a minification pass to remove the redundant conditional.

_webpack.config.js_

```javascript
// define enviromental variable into script files
config.plugins.push(new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),
  __CLIENT__: true,
  __SERVER__: false,
  __DEVELOPMENT__: !production,
  __DEVTOOLS__: !production
}));
```

[extract-text-webpack-plugin](https://github.com/webpack/extract-text-webpack-plugin)  
It moves every require("style.css") in entry chunks into a separate css output file. So your styles are no longer inlined into the javascript, but separate in a css bundle file (styles.css). If your total stylesheet volume is big, it will be faster because the stylesheet bundle is loaded in parallel to the javascript bundle.  

```sh
npm install extract-text-webpack-plugin --save-dev
```
_webpack.config.js_

```javascript
config.plugins = [];
if (production) {
  config.plugins.push(new ExtractTextPlugin('[name].css', {
    allChunks: true
  }));
}
```

[webpack.optimize.UglifyJsPlugin](https://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin)  
Minimize all JavaScript output of chunks. Loaders are switched into minimizing mode. You can pass an object containing UglifyJS options.

_webpack.config.js_

```javascript
const webpack = require('webpack');
config.plugins = [];
if (production) {
  config.plugins.push(new webpack.optimize.UglifyJsPlugin({
    compressor: {
      warnings: false
    }
  }));
}
```

[webpack.optimize.DedupePlugin](https://webpack.github.io/docs/list-of-plugins.html#dedupeplugin)  
Search for equal or similar files and deduplicate them in the output. This comes with some overhead for the entry chunk, but can reduce file size effectively.  
This doesn’t change the module semantics at all. Don’t expect to solve problems with multiple module instance. They won’t be one instance after deduplication.  
Note: Don’t use it in watch mode. Only for production builds.

_webpack.config.js_

```javascript
config.plugins = [];
if (production) {
  config.plugins.push(new webpack.optimize.DedupePlugin());
}
``` 

Other todo

_webpack.config.js_

```javascript
config.plugins = [];
if (production) {
  config.plugins.push(new webpack.optimize.OccurenceOrderPlugin());
  config.plugins.push(new webpack.optimize.LimitChunkCountPlugin({
    maxChunks: 15
  }));
  config.plugins.push(new webpack.optimize.MinChunkSizePlugin({
    minChunkSize: 10000
  }));
}

```

## Babel

### Webpack Babel

This package allows transpiling JavaScript files using Babel and webpack.  
[https://github.com/babel/babel-loader](https://github.com/babel/babel-loader)

```sh
npm install babel-loader --save-dev
```

### Babel core and presets

[https://babeljs.io/docs/setup/](https://babeljs.io/docs/setup/)

```sh
npm install babel-core --save-dev
```

Add preset

```sh
npm install babel-preset-es2015 --save-dev
```

_.babelrc_

```json
{
  "presets": ["es2015"]
}
```
_site.js_

```javascript
'use strict';

import './global.scss';

const div = document.querySelector('.app');
div.innerHTML = 'Hello JS';
console.log('Hello JS!');
```
_webpack.config.js_

```javascript
'use strict';

const path = require('path');
const ExtractTextPlugin = require('extract-text-webpack-plugin');

const production = process.env.NODE_ENV === 'production';
const testing = process.env.NODE_ENV === 'testing';

let config = {
  entry: {
    site: path.join(__dirname, 'src/site.js')
  },
  output: {
    path: path.join(__dirname, 'public/assets'),
    filename: '[name].js'       
  },
  resolve: {
    modulesDirectories: [
      'src',
      'node_modules',
      'bower_components'
    ],
    root: path.resolve('./src/')
  }
}

config.module = {
  loaders: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: production ? 'babel-loader' : 'react-hot!babel-loader'
      },
      {
        test: /\.(scss)$/,
        loader: production
        ? ExtractTextPlugin.extract('css-loader?sourceMap!postcss-loader!sass-loader?sourceMap')
        : 'style-loader!css-loader?sourceMap!postcss-loader!sass-loader?sourceMap'
      },
      {
        test: /\.(css)$/,
        loader: production
          ? ExtractTextPlugin.extract('css-loader?sourceMap!postcss-loader')
          : 'style-loader!css-loader?sourceMap!postcss-loader'
      }
  ]
}

config.plugins = [
  new ExtractTextPlugin('[name].css', { allChunks: true })
];

module.exports = config;
```


### Babel polyfill

[https://babeljs.io/docs/usage/polyfill/](https://babeljs.io/docs/usage/polyfill/)

```sh
npm install babel-polyfill --save-dev
```
_site.js_

```javascript
import 'babel-polyfill';
```

## ESLint

### ESLint


<http://eslint.org/docs/user-guide/configuring#specifying-parser>  
<http://eslint.org/docs/user-guide/configuring#ignoring-files-and-directories>

Isntall ESLint

```sh
npm i -D eslint
```

Isntall needed plugins
```sh
npm i -D eslint babel-eslint eslint-config-standard eslint-plugin-babel eslint-plugin-promise eslint-plugin-react eslint-plugin-standard
```

_.eslintrc_

```json
{
  "extends": ["standard"],

  "ecmaFeatures": {
    "jsx": true
  },
  "plugins": [
    "standard",
    "babel",
    "react"
  ],
  "parser": "babel-eslint",
  "rules": {

    // esling rules
    // Reference: http://eslint.org/docs/rules/
    // instead of using esliny rules we use standard

    // enable back semicolons
    // instead of using semistandard
    "semi": [2, "always"],
    "no-extra-semi": 2,
    "semi-spacing": [2, { "before": false, "after": true }],

    "jsx-quotes": [2, "prefer-double"],

    // babel plugin
    "babel/generator-star-spacing": 1,
    "babel/new-cap": 1,
    "babel/object-curly-spacing": 1,
    "babel/object-shorthand": 1,
    "babel/arrow-parens": 1,

    // react plugin
    // Reference: https://github.com/yannickcr/eslint-plugin-react
    "react/display-name": [2, {"ignoreTranspilerName": false}],
    "react/jsx-boolean-value": [1, "never"],
    "react/jsx-closing-bracket-location": [1, {"location": "tag-aligned"}],
    "react/jsx-curly-spacing": [2, "never"],
    "react/jsx-indent-props": [1, 2],
    "react/jsx-max-props-per-line": [1, {"maximum": 2}],
    "react/jsx-no-duplicate-props": [2, {"ignoreCase": false}],
    "react/jsx-no-literals": 1,
    "react/jsx-no-undef": 2,
    // "react/jsx-sort-prop-types": [1, {"ignoreCase": false}],
    // "react/jsx-sort-props": [1, {"ignoreCase": true}],
    "react/jsx-uses-react": 1,
    "react/jsx-uses-vars": 2,
    "react/no-danger": 1,
    "react/no-multi-comp": 1,
    "react/no-unknown-property": 2,
    "react/prop-types": 2,
    "react/react-in-jsx-scope": 1,
    "react/require-extension": 2,
    "react/self-closing-comp": 2,
    "react/sort-comp": 1,
    "react/wrap-multilines": 2,
    // most of the state should be in your flux implementation, but there
    // are valid reasons for having state in components, just know what
    // your doing
    "react/no-set-state": 0,
    // keeping these as warning, any state changing in these could only happend
    // if an async action is sent, else there willbe layout trashing and
    // multiple render calls
    "react/no-did-mount-set-state": 1,
    "react/no-did-update-set-state": 1
    }
}

```

_.eslintignore_

```
# node_modules/*
# bower_components/*
static/*
```

### Webpack ESLint loader

```sh
npm i -D eslint-loader
```

_webpack.config.js_

```javascript
config.module = {
  preLoaders: [
    {
      test: /\.js$/,
      exclude: /node_modules/,
      loader: 'eslint-loader'
    }
  ]
}

config.eslint = {
  quite: !production,
  failOnWarning: false,
  failOnError: production
};
```

### Sublime Text 3 - ESLint

<https://github.com/roadhump/SublimeLinter-eslint>  
<http://sublimelinter.readthedocs.io/en/latest/installation.html>  
<http://sublimelinter.readthedocs.io/en/latest/settings.html>

```
Cmd+Shift+P
SublimeLinter
SublimeLinter-contrib-eslint
```

Enable elint in project.sublime-project or package settings (default / user)

_project.sublime-project_

```json
"SublimeLinter":
	{
		"linters":
		{
			"eslint":
			{
        		"@disable": false,
        		"args": [],
				"excludes":
				[
					"static/*",
					"node_modules/*",
					"bower_components/*"
				]
			},
			"jshint":
			{
				"@disable": true
			}
		}
	}
```

_SublimeLinter > Settings - User_

```json
{
    "user": {
        "debug": false,
        "delay": 0.25,
        "error_color": "D02000",
        "gutter_theme": "none",
        "gutter_theme_excludes": [],
        "lint_mode": "background",
        "linters": {
            "eslint": {
                "@disable": true,
                "args": [],
                "excludes": []
            },
            "jshint": {
                "@disable": true,
                "args": [],
                "excludes": []
            },
            "pep8": {
                "@disable": true,
                "args": [],
                "disable": "",
                "enable": "",
                "excludes": [],
                "ignore": "",
                "max-line-length": null,
                "rcfile": "",
                "select": ""
            }
        },
        "mark_style": "outline",
        "no_column_highlights_line": false,
        "passive_warnings": false,
        "paths": {
            "linux": [],
            "osx": [
                "/usr/local/bin/"
            ],
            "windows": []
        },
        "python_paths": {
            "linux": [],
            "osx": [
                "/usr/local/bin/"
            ],
            "windows": []
        },
        "rc_search_limit": 3,
        "shell_timeout": 10,
        "show_errors_on_save": false,
        "show_marks_in_minimap": true,
        "syntax_map": {
            "html (django)": "html",
            "html (rails)": "html",
            "html 5": "html",
            "php": "html",
            "python django": "python"
        },
        "warning_color": "DDB700",
        "wrap_find": true
    }
}
```

Set build for ESLint autofix  
<http://sublimetext.info/docs/en/reference/build_systems.html>

`Tools > Build System > New Build System`

```json
{
  "shell_cmd": "$project_path/node_modules/.bin/eslint --fix $file",
  "selector": "*.js"
}
```

Set `Tools > Build System > Automatic` to true. Whenever build (shortcut `Cmd+B`) is called from `.js` file it wil be fix-linted. You can cgange/add global build shortcut in `Preferences > Key Bindings - User`

```json
{
	{ "keys": ["super+b"], "command": "build"}
}
```

### Sublime Text 3 - jshint (simple)

This is just fast error checking, use as reference.
Use SublimeLinter-eslint for linting, so that `.eslintrc` (profiles) and `.eslintignore` are used.

<https://github.com/SublimeLinter/SublimeLinter-jshint>  
<http://sourabhbajaj.com/mac-setup/SublimeText/SublimeLinter.html>

```sh
npm install -g jshint
```

```
Cmd+Shift+P
SublimeLinter
SublimeLinter-jshint
```

Enable jshint in `project.sublime-project` or package settings (default / user)

***

##React.js

### React.js

[https://facebook.github.io/react/downloads.html](https://facebook.github.io/react/downloads.html)

```sh
npm install react --save-dev
npm install react-dom --save-dev
```
_site.js_

```javascript
'use strict';

import 'babel-polyfill';
import './global.scss';

var React = require('react');
var ReactDOM = require('react-dom');

ReactDOM.render(
  <h1>Hello, React.js!</h1>,
  document.querySelector('.app')
);
```

### Webpack hot loader for React

```sh
npm install react-hot-loader --save-dev
```
_webpack.config.js_

```javascript
    {
      test: /\.js$/,
      exclude: /node_modules/,
      loader: production ? 'babel-loader' : 'react-hot-loader!babel-loader'
    },
```


### Babel preset for JSX and ES7+

```sh
npm install babel-preset-react --save-dev
npm install babel-preset-stage-0 --save-dev 
```
_.babelrc_

```json
{
  "presets": ["es2015", "react", "stage-0"],
  "plugins": [
    ["transform-decorators-legacy"]
  ]
}
```
_site.js_

```javascript
'use strict';

import 'babel-polyfill';
import './global.scss';

import React, {Component, PropTypes} from 'react';
import ReactDOM from 'react-dom';

class Counter extends Component {
  static propTypes = {
    initialCount: PropTypes.number
  };
  static defaultProps = {
    initialCount: 0
  };
  constructor (props) {
    super(props);
    this.state = {
      count: this.props.initialCount
    };
    // this.onClickHandler = this.onClickHandler.bind(this);
    this.onClickHandler = ::this.onClickHandler;
  }
  onClickHandler () {
    this.setState({
      count: this.state.count + 1
    });
  }
  render () {
    const {count} = this.state;
    return (
      <div onClick={this.onClickHandler}>
        {`Mouse click count: ${count}`}
      </div>
    );
  }
}

ReactDOM.render(
  <Counter />,
  document.querySelector('.app')
);
```
<https://facebook.github.io/react/blog/2015/01/27/react-v0.13.0-beta-1.html>



### ESLint plugin for React

Done previously

```sh
npm install eslint-plugin-react --save-dev
```

### Sublime Text Edit 3 JSX coments

[CommentJSX-Sublime-Text-3](https://github.com/WARP-LAB/CommentJSX-Sublime-Text-3)

### Classnames

[Classnames](https://github.com/JedWatson/classnames)

```sh
npm install classnames --save-dev
```

### Add-ons

<https://facebook.github.io/react/docs/addons.html>

```
    "react-addons-css-transition-group": "^0.14.7",
```

## Flux & Redux

```sh
npm install redux --save-dev
npm install react-redux --save-dev
npm install react-router --save-dev
npm install react-router-redux --save-dev
npm install history@">=0.0.0 <3.0.0" --save-dev
```

Add `babel-plugin-transform-decorators-legacy` so we can use `@connect` decorator

.babelrc  

```
{
  "presets": ["es2015", "stage-0", "react"],
  "plugins": [
    ["transform-decorators-legacy"]
  ]
}
```

<https://egghead.io/courses/getting-started-with-redux>  
<https://egghead.io/courses/building-react-applications-with-idiomatic-redux>  

