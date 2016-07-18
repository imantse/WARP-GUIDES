# WEBPACK HOWTO

This is a quick and unedited guide how to set up [webpack](https://webpack.github.io) build system in the moment you drop [gulp](http://gulpjs.com).  
Firstly it assumes plain SCSS, JavaScript ES5. Then drops in loaders/plugins for CSS and Javascript ES6/ES2015 (PostCSS, Babel, ESLint a.o.). 
Finally it adds React.js in the mix.

webpack 1.13.x-1.14.x assumed.

Put together by @kroko for the new collegues that see webpack for the first time ;)

Please

* clone it
* read it
* learn it
* create new project in our server and test it step by step
* there are things that are left out (especially server proxy config for `webpack-dev-server`, so ask if you don't see connection from some A to B)
* watch out for errors (lot of stuff here is untested, as I don't have such not-real-world (barebone) code anywhere) as well as things that simply do not work anymore, because there is better way to do it (webpack gets updates, you know...)
* add, commit and push fixes/changes/additions to this repo so that we can make this the ultimate webpack guide.

## Result

### File tree

This howto assumes file tree ~ like this

```
├── package.json
├── public
│   ├── assets
│   └── index.html
├── src
│   ├── components
│   ├── containers
│   ├── fonts
│   ├── hepers
│   ├── images
│   ├── global.scss
│   ├── mixins.scss
│   ├── preflight.js
│   ├── site.js
│   ├── typography.scss
│   └── variables.scss
└── webpack.config.js
```

### Example `public/index.(html|php)`

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>My Title</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <link rel="stylesheet" type="text/css" href="./assets/site.css">
  <script src="./assets/preflight.js"></script>
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

## Vanilla JavaScript and SCSS/CSS setup


### Set up webpack so that we can simply build js file

```sh
npm install webpack --save-dev
```

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

Notice, that entry point __key names__ are being outputed to `./public/assets` as files. That is, you can change key name and real file name to whatever, i.e.,

```javascript
entry: {
  myBundle: './src/whatever.js'
},
```

and you will get `myBundle.js` in `./public/assets` (later you will see that CSS that is imported through JavaScript also will be named as the entry point name, i.e., `./public/assets/myBundle.css`).

You can change this behaviour if output `filename: '[name].js'` is set to `filename: 'someConstantName.js'`. But don't do that, let your entry point key names (and you will have plenty of them in a real world scenario) dictate the output name.

### Webpack minimize JavaScript

[webpack.optimize.UglifyJsPlugin](https://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin)

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

Run webpack, specify ENV

```sh
NODE_ENV=production webpack --progress
```


### node-sass

Node-sass is a library that provides binding for Node.js to LibSass, the C version of the popular stylesheet preprocessor, Sass.  
[https://github.com/sass/node-sass](https://github.com/sass/node-sass)

```sh
npm install node-sass --save-dev
```

### Webpack CSS loaders so that CSS/SCSS can be required in js

[https://github.com/webpack/css-loader](https://github.com/webpack/css-loader)

[https://github.com/jtangelder/sass-loader](https://github.com/jtangelder/sass-loader)

[https://github.com/postcss/postcss-loader](https://github.com/postcss/postcss-loader)

It moves every require("style.css") in entry chunks into a separate css output file. So your styles are no longer inlined into the javascript, but separate in a css bundle file (styles.css).
[extract-text-webpack-plugin](https://github.com/webpack/extract-text-webpack-plugin)

```sh
npm install css-loader --save-dev
npm install sass-loader --save-dev
npm install postcss-loader --save-dev
npm install extract-text-webpack-plugin --save-dev
```

_webpack.config.js_

```javascript
'use strict';

const path = require('path');
const ExtractTextPlugin = require('extract-text-webpack-plugin');

let config = {
  entry: {
    site: path.join(__dirname, 'src/site.js')
  },
  output: {
    path: path.join(__dirname, 'public/assets'),
    filename: '[name].js'       
  }
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
        test: /\.(scss)$/,
        loader: ExtractTextPlugin.extract('css-loader?sourceMap!postcss-loader!sass-loader?sourceMap')
      }
  ]
}

config.plugins = [];

if (production) {
  config.plugins.push(new ExtractTextPlugin('[name].css', {
    allChunks: true
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

### PostCSS plugins

[autoprefixer](https://github.com/postcss/autoprefixer)  
[node-css-mqpacker](https://github.com/hail2u/node-css-mqpacker)  
[cssnano](https://github.com/ben-eb/cssnano)

```sh
npm install css-mqpacker --save-dev
npm install autoprefixer --save-dev
npm install cssnano --save-dev
```
_webpack.config.js_

```javascript
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
      }
    })
  ];
};
```

Alternatives  
[https://github.com/jonathantneal/postcss-time-machine](https://github.com/jonathantneal/postcss-time-machine)

### normalize.css

[https://necolas.github.io/normalize.css/](https://necolas.github.io/normalize.css/)

```sh
npm install normalize.css --save-dev
```
_site.scss_

```scss
@import "~normalize.css";
```

### Webpack CSS source maps

_webpack.config.js_

```javascript
config = {
  devtool: production ? 'source-map' : 'inline-source-map',
  ...
}
```

### Webpack webpack-dev-server

[https://github.com/webpack/webpack-dev-server](https://github.com/webpack/webpack-dev-server)

```sh
npm i -D webpack-dev-server
```

### Webpack style-loader

Adds CSS to the DOM by injecting a `<style>` tag.  Use with webpack-dev-server.
[https://github.com/webpack/style-loader](https://github.com/webpack/style-loader)

```sh
npm i -D style-loader
```

### Webpack file-loader

[https://github.com/webpack/file-loader](https://github.com/webpack/file-loader)

```sh
npm i -D file-loader
```

### Webpack url-loader

The url loader works like the file loader, but can return a Data Url if the file is smaller than a limit.  
[https://github.com/webpack/url-loader](https://github.com/webpack/url-loader)

```sh
npm i -D url-loader
```
_webpack.config.js_

```javascript
    {
      test: /\.(png|jpg|gif)$/,
      loader: production
        ? 'url-loader?limit=10000'
        : 'url-loader?limit=10000?'
    },
```

### Webpack image-webpack-loader

Minify PNG, JPEG, GIF and SVG images with imagemin.  
[https://github.com/imagemin/imagemin](https://github.com/imagemin/imagemin)

```sh
npm i -D image-webpack-loader
```
_webpack.config.js_

```javascript
    {
      test: /\.(png|jpg|gif)$/,
      loader: production
        ? 'url-loader?limit=10000!image-webpack'
        : 'url-loader?limit=10000?'
    },
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

### Webpack Resolve URL Loader

This is needed when include paths from CSS can not be resolved. This is because we write `ulr()` inside SCSS relative to the SCSS file, however Webpack tries tu resolve from entry point (`site.scss`).  
`@font-face` `url()` is obvious example.

```sh
npm install resolve-url-loader --save-dev
```
_webpack.config.js_

```javascript
    {
      test: /\.(scss)$/,
      loader: production
      ? ExtractTextPlugin.extract('css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap')
      : 'style-loader!css-loader?sourceMap!postcss-loader!resolve-url-loader?keepQuery!sass-loader?sourceMap'
    },
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

