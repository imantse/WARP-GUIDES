# WEBPACK BEGINNERS GUIDE <sup>+ npm side notes</sup>

### About

This is a quick, on demand and yet unedited/untested guide how to set up [webpack](https://webpack.github.io) build system (in the moment you drop [gulp](http://gulpjs.com) on when you haven't used any building/packing system before).

Put together by @kroko for the new colleagues that see webpack for the first time. However I tried to formulate things in a way so that other random readers may also benefit.  
_While doing edits I realised that notes should be made for some basic npm stuff as the reality is - there are people who haven't used any building tools or even npm before (not-Node.js backend guys - PHP/ROR/... - leaning towards full stack of frontend), but want to jump in webpack. So this assumes entry level knowledge in Node/NPM._

### Steps

* In the beginning examples will use JavaScript ES3/ES5 and SCSS.
* Then we will drop in loaders/plugins for SCSS ([PostCSS](http://postcss.org) + plugins).
* Then we will look how to set up basic hot-reloading dev server.
* At this point you should be able to code oldschool sites using modern building system - ES3/ES5 JavaScript, use SCSS (I said _oldschool_, writing CSS directly is just _archaic_), hot reloading.
* Then we set up system so that we can code in ES6 as well as check our code ([Babel](https://babeljs.io), [ESLint](http://eslint.org)).
* We add polyfills that we get out of the box from Babel.
* As a sidestep we look how to enable linter for text editors.
* Finally we add [React.js](https://facebook.github.io/react/) in the mix, needed loaders and configuration.

### Assumptions

* webpack 1.13.x-1.14.x.
* Examples are for frontend only. Guide assumes Apache2 server (our devserver pool) for which a virtual host is configured that the `public` directory that you will see later is the `DocumentRoot` of that vhost. Just create new devsite via `warpdevsite nameformywebpacktest && cdd nameformywebpacktest ` and you're all set. If you want to try this outside our devserver with small modifications it will also work if serving stuff via NGINX (proxy in `.htaccess` that I'm going to talk about then should go in conf) as well as Node.js (simple `http` + `node-static` or full Express.js).

### Your task

* clone it
* read it
* learn it
* create new project in our server `warpdevsite` and test it step by step
* there are things that are left out, so ask if you don't see how to get from A to B
* watch out for errors (some stuff here is untested + things that simply do not work anymore, because there is better way to do it and/or webpack gets updates, you know...)
* add, commit and push fixes/changes/additions to this repo so that we can make this the ultimate beginners webpack + npm guide.

---
# PREFLIGHT
---

## SET UP BASIC FILES & DIR STRUCTURE

Crate master directory and set files tree up like this (`tree -a .`).  
Leave all files empty, we will fill them up step by step.  
Remember - `public` is `DocumentRoot`, served at `nameformywebpacktest.our.dev.host.tld`.

```
master-directory
├── package.json
├── public
│   ├── assets
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

## NPM INIT

`cd` in master directory and initialise npm in it. Not needed if you have placed template or DIY `package.json` already there.

```sh
npm init
```

---
# Vanilla JavaScript and SCSS/CSS setup
---

*Note for [absolute beginners](https://www.youtube.com/watch?v=r8NZa9wYZ_U). All `npm` as well as `webpack` commands are executed while being `cd`-ed in this projects 'master directory'. You can do it while being somewhere else via `npm --prefix ${DIRNAME} install ${DIRNAME}` though. RTFM@NPM / ask.*

## Set up webpack so that we can simply build JavaScript file

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
};
module.exports = config;
```

Set up hello world javascript that selects `app` div in our html and puts some text in it. Note that we are using ES5 here.

_src/site.js_

```javascript
'use strict';
var div = document.querySelector('.app');
div.innerHTML = 'Hello JS';
console.log('Hello JS!');
```

Run webpack (before that clean assets directory)

```sh
rm -rf public/assets/** && webpack --progress
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


## Build multiple JavaScript files

Our template has `preflight.js` in the head which is not present after building in previous step (and we get `404`). So let us add new endpoint for preflight.

_src/preflight.js_

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
};
module.exports = config;
```

Run webpack

```sh
rm -rf public/assets/** && webpack --progress
```

Inspect. The file is in `public/assets` directory and it does its job.


## Webpack minimise JavaScript

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
};

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
rm -rf public/assets/** && NODE_ENV=production webpack --progress
```

Inspect how `assets/site.js` changes based on whether `NODE_ENV` is set to `production`.


## Webpack CSS loaders so that CSS/SCSS can be required in JavaScript

### node-sass

For to compile SCSS to CSS we will be using [Node-sass](https://github.com/sass/node-sass). It _is a library that provides binding for Node.js to LibSass, the C version of the popular stylesheet preprocessor, Sass_.  

```sh
npm install node-sass --save-dev
```

### loaders

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
};

// ----------------
// MODULES

config.module = {
  loaders: [
    {
      test: /\.(scss)$/,
      loader: ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!sass-loader?sourceMap')
    }
  ]
};

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

_src/site.js_

```javascript
'use strict';

require('./global.scss');

var div = document.querySelector('.app');
div.innerHTML = 'Hello JS';
console.log('Hello JS!');
```

_src/global.scss_

```scss
.app {
  background-color: red;
}
```

Run webpack and inspect `public/assets/site.css`

```sh
rm -rf public/assets/** && webpack --progress
```

SCSS is compiled and spit out in a file under `public/assets` named the same as the entry point key of the JavaScript from which SCSS was included in the first place.

## Note about loader names

Whenever you use loaders

```javascript
('style-loader', 'css-loader?sourceMap!sass-loader?sourceMap')
```

you can omit `-loader` part

```javascript
('style', 'css?sourceMap!sass?sourceMap')
```

But IMHO it's better to be explicit as it helps finding, batch replacing things.

## PostCSS plugins

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
  // we minimise CSS always as we have source maps, but for this example let us do conditional
  if (production) {
    postPluginConf.push(
      require('cssnano')({
        discardComments: {
          removeAll: true
        },
        autoprefixer: false,
        reduceIdents: false,
        zindex: false
      })
    );
  }
  return postPluginConf;
};
...

```

_src/global.scss_

```scss
.app {
  background-color: red;
  display: flex;
  transform: translateY(50px);
}
```

Run webpack and inspect `public/assets/site.css`

```sh
rm -rf public/assets/** && webpack --progress
```

## normalize.css

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
rm -rf public/assets/** && webpack --progress
```

## Webpack CSS source maps

As you can see we pass `?sourceMap` to our loaders (currently we do not have any loaders for JavaScript). But where are the source maps? Use webpack [devtool](https://webpack.github.io/docs/configuration.html#devtool).

First try

_webpack.config.js_

```javascript
let config = {
  devtool: 'inline-source-map',
  ...
};
```

Run webpack and inspect `public/assets/site.js` and `public/assets/site.css`

```sh
rm -rf public/assets/** && webpack --progress
```

Now try

```javascript
let config = {
  devtool: 'source-map',
  ...
};
```

Run webpack and inspect `public/assets/` directory.

Finally set for now to always generate source maps, as we are testing things here

_webpack.config.js_

```javascript
let config = {
  devtool: production ? 'source-map' : 'inline-source-map',
  ...
};
```

In real world I tend to use

```javascript
devtool: (production) ? null : 'inline-source-map',
```

which would enable source maps both for development and staging.


## Webpack file-loader & url-loader & resolve-url-loader

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

Add loader in the pipe and loader rule to handle this file type (don't be surprised about not including SVG, later on that)

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

## Webpack image-webpack-loader

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

This process is expensive. In development we do not care about file size as we live within gigabit ethernet, so set it to 

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

## Fonts - webfont building

Convert TTF/OTF to all webwonts (WOFF, WOFF2, EOT, SVG) in building process.

Ask.

## Fonts - webfont packing & loading

Let us add new directory `fonts`. Choose font that has few styles (regular and bold + italic) for simplicity. [Space Mono](https://www.fontsquirrel.com/fonts/space-mono)

We need also extra files: `fonts/spacemono-definition.scss`, `src/typography.scss`


```
├── package.json
├── public
│   ├── assets
│   ├── .htaccess
│   └── index.html
├── src
│   ├── fonts
│   │   ├── spacemono
│   │   │   ├── spacemono-bolditalic-webfont.eot
│   │   │   ├── spacemono-bolditalic-webfont.svg
│   │   │   ├── spacemono-bolditalic-webfont.ttf
│   │   │   ├── spacemono-bolditalic-webfont.woff
│   │   │   ├── spacemono-bolditalic-webfont.woff2
│   │   │   ├── spacemono-bold-webfont.eot
│   │   │   ├── spacemono-bold-webfont.svg
│   │   │   ├── spacemono-bold-webfont.ttf
│   │   │   ├── spacemono-bold-webfont.woff
│   │   │   ├── spacemono-bold-webfont.woff2
│   │   │   ├── spacemono-italic-webfont.eot
│   │   │   ├── spacemono-italic-webfont.svg
│   │   │   ├── spacemono-italic-webfont.ttf
│   │   │   ├── spacemono-italic-webfont.woff
│   │   │   ├── spacemono-italic-webfont.woff2
│   │   │   ├── spacemono-regular-webfont.eot
│   │   │   ├── spacemono-regular-webfont.svg
│   │   │   ├── spacemono-regular-webfont.ttf
│   │   │   ├── spacemono-regular-webfont.woff
│   │   │   └── spacemono-regular-webfont.woff2
│   │   └── spacemono-definition.scss
│   ├── global.scss
│   ├── images
│   │   ├── my-large-image.jpg
│   │   └── my-small-image.jpg
│   ├── preflight.js
│   ├── site.js
│   └── typography.scss
└── webpack.config.js
```

Define font family

_fonts/spacemono-definition.scss_

```scss
@font-face {
    font-family: 'spacemono-webpack';
    src: url('./spacemono/spacemono-bold-webfont.eot');
    src: url('./spacemono/spacemono-bold-webfont.eot?#iefix') format('embedded-opentype'),
         url('./spacemono/spacemono-bold-webfont.woff2') format('woff2'),
         url('./spacemono/spacemono-bold-webfont.woff') format('woff'),
         url('./spacemono/spacemono-bold-webfont.ttf') format('truetype'),
         url('./spacemono/spacemono-bold-webfont.svg#space_monobold') format('svg');
    font-weight: 700;
    font-style: normal;
}

@font-face {
    font-family: 'spacemono-webpack';
    src: url('./spacemono/spacemono-bolditalic-webfont.eot');
    src: url('./spacemono/spacemono-bolditalic-webfont.eot?#iefix') format('embedded-opentype'),
         url('./spacemono/spacemono-bolditalic-webfont.woff2') format('woff2'),
         url('./spacemono/spacemono-bolditalic-webfont.woff') format('woff'),
         url('./spacemono/spacemono-bolditalic-webfont.ttf') format('truetype'),
         url('./spacemono/spacemono-bolditalic-webfont.svg#space_monobold_italic') format('svg');
    font-weight: 700;
    font-style: italic;
}

@font-face {
    font-family: 'spacemono-webpack';
    src: url('./spacemono/spacemono-regular-webfont.eot');
    src: url('./spacemono/spacemono-regular-webfont.eot?#iefix') format('embedded-opentype'),
         url('./spacemono/spacemono-regular-webfont.woff2') format('woff2'),
         url('./spacemono/spacemono-regular-webfont.woff') format('woff'),
         url('./spacemono/spacemono-regular-webfont.ttf') format('truetype'),
         url('./spacemono/spacemono-regular-webfont.svg#space_monoregular') format('svg');
    font-weight: 400;
    font-style: normal;
}

@font-face {
    font-family: 'spacemono-webpack';
    src: url('./spacemono/spacemono-italic-webfont.eot');
    src: url('./spacemono/spacemono-italic-webfont.eot?#iefix') format('embedded-opentype'),
         url('./spacemono/spacemono-italic-webfont.woff2') format('woff2'),
         url('./spacemono/spacemono-italic-webfont.woff') format('woff'),
         url('./spacemono/spacemono-italic-webfont.ttf') format('truetype'),
         url('./spacemono/spacemono-italic-webfont.svg#space_monoitalic') format('svg');
    font-weight: 400;
    font-style: italic;
}
```

Import definitions, set typography globals

_src/typography.scss_

```scss
@charset 'UTF-8';
@import "fonts/spacemono-definition";
body{
  font-family: 'spacemono-webpack', 'Comic Sans MS', monospace;
  font-weight: 400;
}
em, i {font-style: italic;}
strong, b {font-weight: bold;}
strong em, strong i, b em, b i, em strong, em b, i strong, i b {font-weight: bold;font-style: italic;}

```

Import typography

_src/global.scss_

```scss
@import "~normalize.css";
@import "typography.scss";

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

Add some tags so we can see that bold and normal weight is working

_src/site.js_

```javascript
'use strict';
require('./global.scss');
var div = document.querySelector('.app');
div.innerHTML = '<h1>Hello JS</h1><p>Lorem ipsum.</p>';
console.log('Hello JS!');
```

Add loaders for font files in webpack config

_webpack.config.js_

```javascript
...
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
...
```

Run webpack and inspect `public/assets/` directory and `public/assets/site.css`.

```sh
rm -rf public/assets/** && webpack --progress
```

## Webpack GLSL shader loader

We write our shaders in separate files.

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
_example.js_

```javascript
  shaderProg = new THREE.ShaderMaterial({
    uniforms,
    vertexShader: require('./shader.vert').join('\n'),
    fragmentShader: require('./shader.frag').join('\n')
  });
```


## Other loaders

### Webpack expose loader

Just for reference

When wanting to put things in global namespace

[expose loader](https://github.com/webpack/expose-loader)  

```
npm install expose-loader --save-dev
```

### Webpack script loader

Just for reference

[script-loader](https://github.com/webpack/script-loader)  

```
npm install script-loader --save-dev
```

### Webpack style-loader

Just for reference. We have already installed this.  
Adds CSS to the DOM by injecting a `<style>` tag. Use with webpack-dev-server.
[https://github.com/webpack/style-loader](https://github.com/webpack/style-loader)

```sh
npm i -D style-loader
```

### Webpack other built in plugins to use

#### Obligatory

* [webpack.DefinePlugin](https://webpack.github.io/docs/list-of-plugins.html#defineplugin)  
Define free variables. Useful for having development builds with debug logging or adding global constants. The values will be inlined into the code which allows a minification pass to remove the redundant conditional.

* [webpack.optimize.DedupePlugin](https://webpack.github.io/docs/list-of-plugins.html#dedupeplugin)  
Search for equal or similar files and deduplicate them in the output. This comes with some overhead for the entry chunk, but can reduce file size effectively.  
This doesn’t change the module semantics at all. Don’t expect to solve problems with multiple module instance. They won’t be one instance after deduplication.  
Note: Don’t use it in watch mode. Only for production builds.


#### Optional

* webpack.optimize.OccurenceOrderPlugin
* webpack.optimize.MinChunkSizePlugin
* webpack.optimize.LimitChunkCountPlugin

_webpack.config.js_

```javascript
// define environmental variables into scripts
config.plugins.push(new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),
  __CLIENT__: true,
  __SERVER__: false,
  __DEVELOPMENT__: !production,
  __DEVTOOLS__: !production
}));

config.plugins = [];
if (production) {
  config.plugins.push(new webpack.optimize.DedupePlugin());
  config.plugins.push(new webpack.optimize.OccurenceOrderPlugin());
  config.plugins.push(new webpack.optimize.LimitChunkCountPlugin({maxChunks: 15}));
  config.plugins.push(new webpack.optimize.MinChunkSizePlugin({minChunkSize: 10000}));
  config.plugins.push(new ExtractTextPlugin('[name].css', {allChunks: true}));
  config.plugins.push(new webpack.optimize.UglifyJsPlugin({compressor: {warnings: false}}));
}

```

---
# Webpack dev server
---

We use hot reloading always. This is something like `watch` in gulp which we abused. But this is better out-of-box, especially for React (hot reloading while keeping state). If you haven't used hot watching/hot reloading before then strap in, then read manual.

## Preflight

### Webpack webpack-dev-server

Documentation  
[webpack-dev-server](https://github.com/webpack/webpack-dev-server)

Install dev server

```sh
npm i -D webpack-dev-server
```

### manage-htaccess

We will need port proxying for this. Instead of proxying within Apache2 vhost (only suitable for dev & small loads) on NGINX proxy (ze best!), this will give us fast proxy on/off in case of Apache within `.htaccess`  
[manage-htaccess](https://github.com/WARP-LAB/manage-htaccess)

```sh
npm install manage-htaccess --save-dev
```

## Basic setup

Note that this will assume that folder `public` is `webroot`, thus this is Apache2. For non-webroot locations this works also (the proxying rules and `publicPath` that you will see later have to be set up bit different). For Node.js served stuff (`index.html` is served via `node-static` or Express) altogether it is again different.

So. Let us try

```sh
rm -rf public/assets/** && webpack-dev-server --config=./webpack.config.js --host=nameformywebpacktest.our.dev.host.tld --port=4000 --history-api-fallback -d --inline --hot
```

Something run. Our `index.html` returns `404` to all assets, cannot find anything in `public/assets/`. Why? `public/assets/` directory is empty, right?

Now try accessing <http://nameformywebpacktest.our.dev.host.tld:4000/site.css>. Gooody good.

Ignore the port for now. We still want the path to be `assets/file.ext`, because our `index.html` refers to resources using this path.  
Update _webpack.config.js_, add `publicPath` key

```javascript
...
  output: {
    path: './public/assets',
    filename: '[name].js',
    publicPath: production ? '//nameformywebpacktest.our.dev.host.tld/assets/' : 'http://nameformywebpacktest.our.dev.host.tld/assets/'
  },
...
```

_If you have used webpack-dev-server before, then you know that this can also be managed via `--content-base`_

Kill `ctrl+c` devserver. Rerun again with the same command.  
<http://nameformywebpacktest.our.dev.host.tld:4000/site.css> is 404  
<http://nameformywebpacktest.our.dev.host.tld:4000/assets/site.css> is there
Just as we want it

Now our `index.html` looks for assets under `http://nameformywebpacktest.our.dev.host.tld:80/assets/file.ext`, but it is served under `http://nameformywebpacktest.our.dev.host.tld:4000/assets/file.ext`. Let us proxy the ports, so that when assets is asked through port 80, give assets that are on port 4000.

_public/.htaccess_

```
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews
    </IfModule>
    RewriteEngine On
    RewriteRule ^assets/(.+) http://nameformywebpacktest.our.dev.host.tld:4000/assets/$1 [P,L]
</IfModule>
```

Kill `ctrl+c` devserver. Rerun again with the same command. Open up our page. Should contain `Hello JS` and your beautiful background images.

Fire up _src/global.scss_ and change height of app div, observe browser.

```scss
...
  height: 200px;
...
```
Now kill `ctrl+c` devserver. 

## Ping pong

Rerun statically

```sh
rm -rf public/assets/** && webpack --progress
```

:( page does not work any more, because now files are back in `public/assets`, 4000 port is empty. So we need to be able to turn the proxy on/off in `.htaccess` based on whether we run dynamically hot reloading or run _statically_.

Add `manage-htaccess` in the mix.

_webpack.config.js_

```javascript
...
const production = process.env.NODE_ENV === 'production';

const webpackHtaccess = require('manage-htaccess');
webpackHtaccess(
  [
    {
      tag: 'TESTTAG',
      enabled: !production
    }
  ],
  path.join(__dirname, 'public/.htaccess')
);
...
```

Wrap our proxy within tag

_public/.htaccess_

```
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews
    </IfModule>
    RewriteEngine On
#%!<TESTTAG>
    RewriteRule ^assets/(.+) http://nameformywebpacktest.our.dev.host.tld:4000/assets/$1 [P,L]
#%!</TESTTAG>
</IfModule>

```

How it works:
Whenever `NODE_ENV` is set to `production` _manage-htaccess_ disables anything between the specified tags (there can be multiple occurrences as well as multiple tags) in `.htaccess` file (TESTTAG state is `!production === false`). So our proxy RewriteRule is off. If `NODE_ENV` is not `production` (thus assuming development, whilst actually there should also be staging settings), then anything between the tags is enabled (TESTTAG state is `!production === true`).


So test it now

```sh
rm -rf public/assets/** && webpack-dev-server --config=./webpack.config.js --host=nameformywebpacktest.our.dev.host.tld --port=4000 --history-api-fallback -d --inline --hot
```

```sh
rm -rf public/assets/** && NODE_ENV=production webpack --progress
```

## Dynamic CSS

When running dev server we actually want CSS to be inlined within JavaScript (which will result in `404` for `assets/site.css` and FOUC) so that hot reloading works better. Thus final loader config would look like this. And, yes, add loader to process plain CSS files as they might show up somewhere.

_webpack.config.js_

```javascript
...
    {
      test: /\.(scss)$/,
      loader: production
      ? ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap')
      : 'style-loader!css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap'
    },
    {
      test: /\.(css)$/,
      loader: production
        ? ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!postcss-loader')
        : 'style-loader!css-loader?sourceMap!postcss-loader'
    },
...
```

## Make SCSS environment aware

Our SCSS should also know what ebvironment it is built for.

_webpack.config.js_

```javascript
...
// make environment available as sass variable in SCSS
config.sassLoader = {
  data: `$env: ${process.env.NODE_ENV};`
};
...
```

Usage example

_whatever.scss_


```scss
  @if $env == 'production' {
    background-color: transparent;
  } @else {
    background-color: blue;
  }
```

---
# Use npm `scripts`!
---

It is hard to remember all the commands that need to be executed to run stuff. Therefore always make shortcuts. IMHO `package.json` scripts section should reflect what this package can do!

```json
{
  "name": "webpack-guide",
  "version": "0.0.0",
  "description": "webpack guide for WARP coders",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "run:clean": "rm -rf ./public/assets/**",
    "run:prod": "npm run run:clean && NODE_ENV=production webpack --progress",
    "run:dev": "npm run run:clean && webpack-dev-server --config=./webpack.config.js --host=nameformywebpacktest.our.dev.host.tld --port=4000 --history-api-fallback -d --inline --hot",
    "screen:start": "npm run screen:stop && screen -S webpack-guide -d -m npm run run:dev",
    "screen:enter": "screen -r webpack-guide",
    "screen:stop": "screen -S webpack-guide -X quit 2>/dev/null || :"
  },
  "keywords": [
    "webpack"
  ],
  "author": "kroko",
  "license": "MIT",
  "devDependencies": {
    "autoprefixer": "6.3.7",
    "css-loader": "0.23.1",
    "css-mqpacker": "5.0.1",
    "cssnano": "3.7.3",
    "extract-text-webpack-plugin": "1.0.1",
    "file-loader": "0.9.0",
    "image-webpack-loader": "2.0.0",
    "manage-htaccess": "1.0.0",
    "node-sass": "3.8.0",
    "normalize.css": "4.2.0",
    "postcss-loader": "0.9.1",
    "resolve-url-loader": "1.6.0",
    "sass-loader": "4.0.0",
    "style-loader": "0.13.1",
    "url-loader": "0.5.7",
    "webpack": "1.13.1",
    "webpack-dev-server": "1.14.1"
  }
}

```

* `npm run run:prod` to run production

* `npm run run:dev ` to run dev-server

	* `npm run sreen:start` to start dev-server in a separate screen  
	* `npm run screen:stop` to stop dev-server in that separate screen
	* `npm run screen:enter` to attach to the running screen so you can inspect building errors (`ctrl+a` + `ctrl+d` to detach without interrupting, `ctrl+a` + `:quit` to kill it while being within screen session).

---
# Babel
---

Until now `site.js` contained old javascript (except for Common.js `require()` that got handled by webpack loader). You will have to write ES2015 code with [stage-0](https://tc39.github.io/process-document/) features.

## Babel core and presets

[DOCS](https://babeljs.io/docs/setup/)

Install core

```sh
npm install babel-core --save-dev
```

Install presets

```sh
npm install babel-preset-es2015 --save-dev
npm install babel-preset-stage-0 --save-dev
```

Crete new file _.babelrc_ under master directory and fill it

```json
{
  "presets": ["es2015", "stage-0"]
}
```

## Webpack Babel

This loader allows transpiling JavaScript files using Babel and webpack.  
[babel-loader](https://github.com/babel/babel-loader)

```sh
npm install babel-loader --save-dev
```

So let us make subtle changes in `site.js`. For test use arrow function that is ES6 feature.

_src/site.js_

```javascript
'use strict';
import './global.scss';
const myArrowFunction = () => {
  const div = document.querySelector('.app');
  div.innerHTML = '<h1>Hello JS</h1><p>Lorem ipsum.</p>';
  console.log('Hello JS!');
};
myArrowFunction();
```

Now specify loader for JavaScript files. Exclude `node_modules` as they *should be* ES5 already. Exclude `src/preflight.js` as it by it's role in the webapp should never contain anything ES5+ (ES3 recommended).

_webpack.config.js_

```javascript
    {
      test: /\.js$/,
      exclude: [/node_modules/, /preflight\.js$/],
      loader: 'babel-loader'
    },
```

Build it. Inspect how array function got compiled to old JavaScript, so that oldish browsers do what they are told to.

```javascript
...
	var myArrowFunction = function myArrowFunction() {
	  var div = document.querySelector('.app');
	  div.innerHTML = '<h1>Hello JS</h1><p>Lorem ipsum.</p>';
	  console.log('Hello JS!');
	};
	myArrowFunction();
...
```

## Babel polyfill

As a side note - we need also polyfills. There are so many polyfills out there and methods to polyfill (user agent based), but let us use one supplied by Babel.

[DOCS](https://babeljs.io/docs/usage/polyfill/)

```sh
npm install babel-polyfill --save-dev
```

_src/site.js_

```javascript
'use strict';
import 'babel-polyfill';
import './global.scss';
const myArrowFunction = () => {
  const div = document.querySelector('.app');
  div.innerHTML = '<h1>Hello JS</h1><p>Lorem ipsum.</p>';
  div.classList.add('some-class');
  console.log('Hello JS!');
};
myArrowFunction();
```

Run and inspect `public/site.js`. [Polyfills everywhere](https://cdn.meme.am/instances/500x/65651431.jpg).

---
# ESLint
---

Apart from writing modern JavaScript you will have to obey syntax rules as well as formatting rules. Oh well. __*Tabbing in our source code should be SPACES and tab width is 2 SPACES. Configure your text editor.*__

## ESLint


<http://eslint.org/docs/user-guide/configuring#specifying-parser>  
<http://eslint.org/docs/user-guide/configuring#ignoring-files-and-directories>

Install ESLint

```sh
npm i -D eslint
```

Install plugin for Babel

```sh
npm i -D babel-eslint eslint-plugin-babel
```

Install ESLint plugins

```sh
npm i -D eslint-plugin-promise eslint-plugin-standard
```

Install config we will be using
```sh
npm i -D eslint-config-standard
```

Crete new file _.eslintrc_ under master directory and fill it

```json
{
  "extends": ["standard"],
  "plugins": [
    "standard",
    "babel"
  ],
  "parser": "babel-eslint",
  "rules": {
    "semi": [2, "always"],
    "no-extra-semi": 2,
    "semi-spacing": [2, { "before": false, "after": true }],

    "babel/generator-star-spacing": 1,
    "babel/new-cap": 1,
    "babel/object-curly-spacing": 1,
    "babel/object-shorthand": 1,
    "babel/arrow-parens": 1,
  }
}

```


Crete new file _.eslintignore_ under master directory and fill it

```
# by default: node_modules/*
# by default: bower_components/*
static/*
```

## Webpack ESLint loader

Wee add new loader to our webpack. It is _preloader_, thus it pre-lints our JavaScript files.

```sh
npm i -D eslint-loader
```

Update _webpack.config.js_, add `preLoaders` key to `config.module` object.  
And add ESLint configuration. It will be verbose when `!production` and it will fail on any errors when `production`.

```javascript
config.module = {
  preLoaders: [
    {
      test: /\.js$/,
      exclude: /node_modules/,
      loader: 'eslint-loader'
    }
  ],
  loaders: [
    {...}
  ...
}; 
...

// ----------------
// PLUGINS

config.eslint = {
  quite: !production,
  failOnWarning: false,
  failOnError: production
};
...
```

In _src/site.js_ do something questionable

```javascript
'use strict';
import 'babel-polyfill';
import './global.scss';
const myArrowFunction = () => {
  const div = document.querySelector('.app');
  div.innerHTML = '<h1>Hello JS</h1><p>Lorem ipsum.</p>';
  div.classList.add('some-class');
  console.log('Hello JS!') // <-- like not adding semicolon
};
myArrowFunction();
```

Build it `npm run run:prod`.

Observe webpack building notices/warnings/errors. It should contain something like this

```
ERROR in ./src/site.js
/path/to/site.js
  7:27  error  Missing semicolon  semi
✖ 1 problem (1 error, 0 warnings)
```

Add back semicolon, rebuild, observe.

Do same with `npm run run:dev` and fix error on the fly.


## ESLint in text editors

### Atom

To do

### Sublime Text 3 - ESLint

If you haven't installed package control in Sublime do it  
<https://packagecontrol.io/installation>

Install needed packages

`Cmd+Shift+P` in Sublime and `Package Control: Install Package`

Search and add package `SublimeLinter-contrib-eslint`

<https://github.com/roadhump/SublimeLinter-eslint>  
<http://sublimelinter.readthedocs.io/en/latest/installation.html>  
<http://sublimelinter.readthedocs.io/en/latest/settings.html>

Enable ESLint in `project.sublime-project` 

_project.sublime-project_

```json
{
	"SublimeLinter":
	{
		"linters":
		{
			"eslint":
			{
				"@disable": false,
				"args":
				[
				],
				"excludes":
				[
					"static/*",
					"node_modules/*",
					"bower_components/*",
					"*.php",
					"*.html"
				]
			},
			"jshint":
			{
				"@disable": true
			}
		}
	}
}
```

Or package settings (default / user)  
_Sublime Text > Preferences > Package Settings > SublimeLinter > Settings - User_
which would look something like this

```json
{
    "user": {
        "debug": false,
        "delay": 0.25,
        "error_color": "D02000",
        "gutter_theme": "Packages/SublimeLinter/gutter-themes/Default/Default.gutter-theme",
        "gutter_theme_excludes": [],
        "lint_mode": "background",
        "linters": {
            "eslint": {
                "@disable": false,
                "args": [],
                "excludes": [
    					"static/*",
    					"node_modules/*",
    					"bower_components/*",
    					"*.php",
    					"*.html"
                ]
            },
            "jshint":
			  {
				  "@disable": true
			  }
        },
        "mark_style": "outline",
        "no_column_highlights_line": false,
        "passive_warnings": false,
        "paths": {
            "linux": [],
            "osx": [],
            "windows": []
        },
        "python_paths": {
            "linux": [],
            "osx": [],
            "windows": []
        },
        "rc_search_limit": 3,
        "shell_timeout": 10,
        "show_errors_on_save": true,
        "show_marks_in_minimap": true,
        "syntax_map": {
            "html (django)": "html",
            "html (rails)": "html",
            "html 5": "html",
            "javascript (babel)": "javascript",
            "magicpython": "python",
            "php": "html",
            "python django": "python",
            "pythonimproved": "python"
        },
        "warning_color": "DDB700",
        "wrap_find": true
    }
}
```

Note that we disable `jshint` if it was previously enabled. jshint is just fast basic error checking, too basic. Use ESLint for linting.

Now do some errors in _src/site.js_ and observe how Sublime reports them on the fly. **Note - Atom wins for this feature.**

## Set ESLint autofix in text editors

**Don't trust autofix, use with care, per one file only! This is like autorouting in EDA.. sad panda.**

### Atom

To do

### Sublime

We can set it up via [build system](http://sublimetext.info/docs/en/reference/build_systems.html)

`Tools > Build System > New Build System`

```json
{
  "shell_cmd": "$project_path/node_modules/.bin/eslint --fix $file",
  "selector": "*.js"
}
```

Set `Tools > Build System > Automatic` to true. Whenever build (shortcut `Cmd+B`) is called from `.js` file it wil be fix-linted. You can change/add global build shortcut in `Preferences > Key Bindings - User`

```json
{
	{"keys": ["super+b"], "command": "build"}
}
```

## Set ESLint `script`

**Don't trust autofix, use with care, per one file only! This is like autorouting in EDA.. sad panda.**

Add new script to _package.json_

```json
"lint": "eslint ./src --fix"
```

so that `npm run lint` will autofix all `src` directory.

---
# stylelint
---

Add linting also to your SCSS.

## Webpack stylelint

[stylelint-webpack-plugin](http://stylelint.io/user-guide/complementary-tools/#build-tool-plugins)

```sh
npm install stylelint-webpack-plugin --save-dev
```

_This webpack plugin will also install `stylelint` dependency. Might not be the latest, but whatever._

Create _.stylelintrc_ and fill in some general values. See [documentation](https://github.com/stylelint/stylelint/blob/master/docs/user-guide/configuration.md).

```json
{
  "rules": {
    "block-no-empty": null,
    "color-no-invalid-hex": true,
    "comment-empty-line-before": [ "always", {
      "ignore": ["stylelint-commands", "between-comments"],
    } ],
    "declaration-colon-space-after": "always",
    "indentation": [2, {
      "except": ["value"],
      "severity": "warning"
    }],
    "max-empty-lines": 2,
    "rule-nested-empty-line-before": [ "always", {
      "except": ["first-nested"],
      "ignore": ["after-comment"],
    } ],
    "unit-whitelist": ["px", "em", "rem", "%", "s"]
  }
}
```

Add plugin to _webpack.config.js_

```javascript
...
const StyleLintPlugin = require('stylelint-webpack-plugin');
...
config.plugins = [];
...
config.plugins.push(new StyleLintPlugin({
  configFile: '.stylelintrc',
  files: '**/*.s?(a|c)ss',
  failOnError: false
}));
...
```

Build it, fix SCSS if needed.

## stylelint in text editors

### Atom

To do

### Sublime Text 3 - ESLint

If you haven't installed package control in Sublime do it  
<https://packagecontrol.io/installation>

Install needed packages

`Cmd+Shift+P` in Sublime and `Package Control: Install Package`

Search and add package `SublimeLinter-contrib-stylelint`

If you have valid _.stylelintrc_ set up and installed `stylelint` (again - don't install directly, just install the plugin) in the project, everything should work out of box.

Same as with ESLint - configure general behaviour via _project.sublime-project_ or 
_Sublime Text > Preferences > Package Settings > SublimeLinter > Settings - User_.

---
# React.js
---

## React.js

Install

```sh
npm install react --save-dev
npm install react-dom --save-dev
```

Install Babel preset for React.js & JSX

```sh
npm install babel-preset-react --save-dev
```

Reconfigure _.babelrc_ to include react

```json
{
  "presets": ["es2015", "stage-0", "react"]
}
```

Install ESLint plugin for React.js

```sh
npm install eslint-plugin-react --save-dev
```

Reconfigure _.eslintrc_

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
    "semi": [2, "always"],
    "no-extra-semi": 2,
    "semi-spacing": [2, { "before": false, "after": true }],

    "jsx-quotes": [2, "prefer-double"],

    "babel/generator-star-spacing": 1,
    "babel/new-cap": 1,
    "babel/object-curly-spacing": 1,
    "babel/object-shorthand": 1,
    "babel/arrow-parens": 1,

    "react/display-name": [2, {"ignoreTranspilerName": false}],
    "react/jsx-boolean-value": [1, "never"],
    "react/jsx-closing-bracket-location": [1, {"location": "tag-aligned"}],
    "react/jsx-curly-spacing": [2, "never"],
    "react/jsx-indent-props": [1, 2],
    "react/jsx-max-props-per-line": [1, {"maximum": 2}],
    "react/jsx-no-duplicate-props": [2, {"ignoreCase": false}],
    "react/jsx-no-literals": 1,
    "react/jsx-no-undef": 2,

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

    "react/no-set-state": 0,

    "react/no-did-mount-set-state": 1,
    "react/no-did-update-set-state": 1,
  }
}
```


_src/site.js_

```javascript
'use strict';

import 'babel-polyfill';
import './global.scss';

import React from 'react';
import ReactDOM from 'react-dom';

ReactDOM.render(
  <div><h1>{'Hello, React.js!'}</h1><p>{'Lorem ipsum'}</p></div>,
  document.querySelector('.app')
);
```

Build it

## Webpack hot loader for React

Hot reloading React.js components without messing up the state.

```sh
npm install react-hot-loader --save-dev
```
edit JavaScript loader in _webpack.config.js_

```javascript
    {
      test: /\.js$/,
      exclude: /node_modules/,
      loader: production ? 'babel-loader' : 'react-hot-loader!babel-loader'
    },
```

## React.js getting started code sample

Test it.

_src/site.js_

```javascript
'use strict';

/* global __DEVELOPMENT__ */

import 'babel-polyfill';
import './global.scss';

import React, {Component, PropTypes} from 'react';
import ReactDOM from 'react-dom';

// enable debug via webpack.DefinePlugin
if (__DEVELOPMENT__) {
  window.React = React;
}

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
      <div onClick={this.onClickHandler} style={{width: '100%', height: '100%'}}>
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


## Flux & Redux

Some extra notes about `transform-decorators-legacy` when using Redux.

```sh
npm install redux --save-dev
npm install react-redux --save-dev
npm install react-router --save-dev
npm install react-router-redux --save-dev
npm install history@">=0.0.0 <3.0.0" --save-dev
```

Add `babel-plugin-transform-decorators-legacy` so we can use `@connect` a.o. decorators

```sh
npm install babel-plugin-transform-decorators-legacy --save-dev
```

_.babelrc_

```
{
  "presets": ["es2015", "stage-0", "react"],
  "plugins": [
    ["transform-decorators-legacy"]
  ]
}
```

---
# Result
---

If you add also some other plugins from built in `webpack.optimize.` family the final webpack configuration in this tutorial looks like this.


```javascript
'use strict';
const path = require('path');
const webpack = require('webpack');
const ExtractTextPlugin = require('extract-text-webpack-plugin');
const StyleLintPlugin = require('stylelint-webpack-plugin');

const production = process.env.NODE_ENV === 'production';
const testing = process.env.NODE_ENV === 'testing';

const webpackHtaccess = require('manage-htaccess');
webpackHtaccess(
  [
    {
      tag: 'TESTTAG',
      enabled: !production
    }
  ],
  path.join(__dirname, 'public/.htaccess')
);

// ----------------
// BASE CONFIG

let config = {
  devtool: production ? 'source-map' : 'inline-source-map',
  target: testing ? 'node' : 'web',
  context: __dirname,
  entry: {
    site: './src/site.js',
    preflight: './src/preflight.js'
  },
  output: {
    path: './public/assets',
    filename: '[name].js',
    publicPath: production ? '//nameformywebpacktest.our.dev.host.tld/assets/' : 'http://nameformywebpacktest.our.dev.host.tld/assets/'
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

// ----------------
// MODULES

config.module = {
  preLoaders: [
    {
      test: /\.js$/,
      exclude: /node_modules/,
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
      ? ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap')
      : 'style-loader!css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap'
    },
    {
      test: /\.(css)$/,
      loader: production
        ? ExtractTextPlugin.extract('style-loader', 'css-loader?sourceMap!postcss-loader')
        : 'style-loader!css-loader?sourceMap!postcss-loader'
    },
    {
      test: /\.(png|jpg|jpeg|gif)$/,
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

config.sassLoader = {
  data: `$env: ${process.env.NODE_ENV};`
};

// ----------------
// PLUGINS

config.plugins = [];

config.plugins.push(new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),
  __CLIENT__: true,
  __SERVER__: false,
  __DEVELOPMENT__: !production,
  __DEVTOOLS__: !production
}));

if (production) {
  config.plugins.push(new webpack.optimize.DedupePlugin());
  config.plugins.push(new webpack.optimize.OccurenceOrderPlugin());
  config.plugins.push(new webpack.optimize.LimitChunkCountPlugin({maxChunks: 15}));
  config.plugins.push(new webpack.optimize.MinChunkSizePlugin({minChunkSize: 10000}));
  config.plugins.push(new ExtractTextPlugin('[name].css', {allChunks: true}));
  config.plugins.push(new webpack.optimize.UglifyJsPlugin({compressor: {warnings: false}}));
}

// ----------------
// 3rd party loader and plugin configuration

config.plugins.push(new StyleLintPlugin({
  configFile: '.stylelintrc',
  files: '**/*.s?(a|c)ss',
  failOnError: false
}));

config.eslint = {
  quite: !production,
  failOnWarning: false,
  failOnError: production
};

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
  postPluginConf.push(
	require('cssnano')({
	  discardComments: {
	    removeAll: true
	  },
	  autoprefixer: false,
	  reduceIdents: false,
	  zindex: false
	})
  );
  return postPluginConf;
};

module.exports = config;

```


