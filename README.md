![Logo](docs/meteor-desktop.png)

# Meteor Desktop <sup>beta version<sup>
###### aka Meteor Electron Desktop Client
> Build desktop apps with Meteor & Electron. Full integration with hot code push implementation.

[![npm version](https://img.shields.io/npm/v/meteor-desktop.svg)](https://npmjs.org/package/meteor-desktop)
<sup>Travis</sup> [![Travis Build Status](https://travis-ci.org/wojtkowiak/meteor-desktop.svg?branch=master)](https://travis-ci.org/wojtkowiak/meteor-desktop)
<sup>AppVeyor</sup> [![Build status](https://ci.appveyor.com/api/projects/status/mga230i3avit8ljv/branch/master?svg=true)](https://ci.appveyor.com/project/wojtkowiak/meteor-desktop)
<sup>CircleCI</sup> [![CircleCI](https://circleci.com/gh/wojtkowiak/meteor-desktop/tree/master.svg?style=svg)](https://circleci.com/gh/wojtkowiak/meteor-desktop/tree/master)

![Demo](docs/demo.gif)

## What is this?

This is a complete implementation of integration between `Meteor` and `Electron` aiming to achieve the same level of developer experience like `Meteor` gives. 
To make it clear from the start, this is a **desktop client** - it is just like your mobile 
clients with `Cordova` - but this is for desktops with `Electron`. It also features a full hot code 
push implementation - which means you can release updates the same way you are used to.  

## Prerequisites

 - Meteor >= `1.3.4`<sup>__*1__</sup>
 - at least basic [Electron](http://electron.atom.io/) framework knowledge
 - mobile platform added to project<sup>__*2__</sup>  

<sup>__*1__ `1.3.3` is supported if you will install `meteor-desktop` with `npm >= 3`</sup>

<sup>__*2__ you can always build with `--server-only` if you do not want to have mobile clients,  you do not actually have to have android sdk or xcode to go on with your project</sup>

### Quick start
```bash
 cd /your/meteor/app
 meteor npm install --save-dev meteor-desktop
 # you need to have any mobile platform added (ios/android)
 meteor --mobile-server=127.0.0.1:3000
 
 # open new terminal

 npm run desktop -- init
 npm run desktop

 # or in one command `npm run desktop -- --scaffold` 
```

## Usage `--help`

```
Usage: meteor-desktop [command] [options]
  Commands:

    init                       scaffolds .desktop dir in the meteor app
    run [ddp_url]              (default) builds and runs desktop app
    build [ddp_url]            builds your desktop app
    build-installer [ddp_url]  creates the installer
    just-run                   alias for running `electron .` in `.meteor/desktop-build`
    package [ddp_url]          runs electron packager
    init-tests-support         prepares project for running functional tests of desktop app

  Options:

    -h, --help                            output usage information
    -b, --build-meteor                    runs meteor to obtain the mobile build, kills it after
    -t, --build-timeout <timeout_in_sec>  timeout value when waiting for meteor to build, default 600sec
    -p, --port <port>                     port on which meteor is running, when with -b this will be passed to meteor when obtaining the build
    --production                          builds meteor app with the production switch, uglifies contents of .desktop, packs app to app.asar
    -a, --android                         force add android as a mobile platform instead of ios
    -s, --scaffold                        will scaffold .desktop if not present
    --ia32                                generate 32bit installer
    --win                                 generate also a Windows installer on Mac
    -V, --version                         output the version number

  [ddp_url] - pass a ddp url if you want to use different one than used in meteor's --mobile-server
              this will also work with -b
```
#### `--build-meteor`
If you just want to build the desktop app, package it or build installer without running the 
`Meteor` project separately you can just use `-b` and all will be done automatically - this is useful when 
for example building on a CI etc.
  
#### `--android`
When there is no mobile platform in the project and `-b` is used, mobile platform is added 
automatically and removed at the end of the build process. Normally an `ios` platform is added 
but you can change this to `android` through this option.
   
Documentation
=================
  * [Architecture](#architecture)
     * [How does this work with Meteor?](#how-does-this-work-with-meteor)
     * [How the Electron app is structured?](#how-the-electron-app-is-structured)
  * [Scaffolding your desktop app](#scaffolding-your-desktop-app)
     * [settings.json](#settingsjson)
        * [Applying different window options for different OS](#applying-different-window-options-for-different-os)
        * [Supported dependency version types](#supported-dependency-version-types)
     * [desktop.js](#desktopjs)
        * [skeletonApp](#skeletonapp)
        * [eventsBus](#eventsbus)
        * [modules](#modules)
        * [Module](#module)
  * [Writing modules](#writing-modules)
     * [extract](#extract)
  * [Hot code push support](#hot-code-push-support)
  * [Meteor.isDesktop](#meteorisdesktop)
  * [<code>Desktop</code> and <code>Module</code>](#desktop-and-module)
     * [Module - desktop side](#module---desktop-side)
     * [Desktop - Meteor side](#desktop---meteor-side)
  * [desktopHCP - .desktop hot code push](#desktophcp---desktop-hot-code-push)
     * [How this works](#how-this-works)
     * [Caveats](#caveats)
  * [How to write plugins](#how-to-write-plugins)
     * [meteorDependencies in <code>package.json</code>](#meteordependencies-in-packagejson)
        * [List of known plugins:](#list-of-known-plugins)
  * [Squirrel autoupdate support](#squirrel-autoupdate-support)
  * [Native modules support](#native-modules-support)
  * [Devtron](#devtron)
  * [Testing desktop app and modules](#testing-desktop-app-and-modules)
  * [MD_LOG_LEVEL](#md_log_level)
  * [Packaging](#packaging)
  * [Building installer](#building-installer)
  * [Roadmap](#roadmap)
  * [Contribution](#contribution)
  * [FAQ](#faq)
  * [Changelog](#changelog)

## Architecture

If you have ever been using any `Cordova` plugins before you will find this approach alike. In `Cordova` every plugin exposes its native code through a JS api available in some global namespace like `cordova.plugins`. The approach used here is similar.

In `Electron` app, there are two processes running along in your app. The so-called `main 
process` and `renderer process`. Main process is just a JS code executed in `node`, and the 
renderer is a `Chromium` process. In this integration your `Meteor` app is being run in the 
`renderer` process and your desktop specific code runs in the `main` process. They are 
communicating through IPC events. Basically, the desktop side publishes its API as an IPC event 
listeners. In your `Meteor` code, calling it is as simple as `Desktop.send('module', 'event');`.  

Code on the desktop side is preferred to be modular - that is just for simplifying testing and 
encapsulating functionalities into independent modules. However, you do not have to follow this style, there is an `import` dir in which you can structure your code however you want. The basics of an `Electron` app are already in place (reffered as `Skeleton App`) and your code is loaded like a plugin to it.

Below is a high level architecture diagram of this integration.

![High level architecture](docs/high-level-arch.png)

#### How does this work with Meteor?
> <sup>or how hacky is this?</sup>

The main goal was to provide a non hacky integration without actually submitting any desktop 
oriented pull request to `Meteor`.
The whole concept is based on taking the `web.cordova` build, modifying it as little as possible 
and running it in the `Electron`'s renderer process. The current `cordova` integration 
architecture is more or less conceptually replicated. 

Currently the only modification that the mobile build is subjected to is injecting the `Meteor.isDesktop` variable. 
 
To obtain the mobile build, this integration takes the build from either
`.meteor/local/cordova-build` (version `< 1.3.4.1`) or from `.meteor/local/build/programs/web.cordova`.
Because `index.html` is not present in the `web.cordova` directory and `program.json` lacks 
`version` field, they are just downloaded from the running project.

#### How the `Electron` app is structured?

The produced `Electron` app consists barely of 4 files:

- `app.asar` - bundled `Skeleton App` and `node_modules` (including all your dependencies from 
`settings.json` and modules)
- `meteor.asar` - your `Meteor` app bundled to an `.asar`
- `desktop.asar` - processed contents for `.destkop`
- `package.json` - `Electron` requires a `package.json` to be present

## Scaffolding your desktop app

If you have not run the example from the Quick start paragraph, first you need to scaffold a `.desktop` dir in which your `Electron`'s main process code lives.
To do that run: (assuming `npm install --save-dev meteor-desktop` did add successfully a `desktop` 
entry in the `package.json scripts` section)
```bash
npm run desktop -- init
```

This will generate an exemplary `.desktop` dir. Lets take a look what we can find there:
```
.desktop
├── assets                     # place all your assets here
├── import                     # all code you do not want to structure into modules  
├── modules                    # your desktop modules (check modules section for explanation)
│    └── example               # module example
│         ├── index.js         # entrypoint of the example module
│         ├── example.test.js  # functional test for the example module
│         └── module.json      # module configuration  
├── desktop.js                 # your Electron main process entry point - treated like a module
├── desktop.test.js            # functional test for you desktop app
├── settings.json              # your app settings
└── squirrelEvents.js          # handling of squirrel.windows events
```

Tak a look into the files. Most of them have meaningful comments inside.

Some files are described more in detail below..
### settings.json

This is the main configuration file for your desktop app.
Below you can find brief descriptions of the fields. 

field|description
-----|-----------
`name`|just a name for your project
`version`|version of the desktop app
`projectName`|this will be used as a `name` in the generated app's package.json
`devTools`|whether to install and open `devTools`, set automatically to false when building with `--production`
`devtron`|check whether to install [`devtron`](http://electron.atom.io/devtron/), set automatically to false when building with `--production`, [more](#devtron)
`desktopHCP`|whether to use `.desktop` hot code push module - [more](#desktophcp---desktop-hot-code-push)
<sup>`desktopHCPIgnoreCompatibilityVersion`</sup>|ignore the `.desktop` compatibility version and install new versions even if they can be incompatible
`autoUpdateFeedUrl`|url passed to [`autoUpdater.setFeedUrl`](https://github.com/electron/electron/blob/master/docs/api/auto-updater.md#autoupdatersetfeedurlurl-requestheaders), [more](#squirrel-autoupdate-support)
`autoUpdateFeedHeaders`|http headers passed to [`autoUpdater.setFeedUrl`](https://github.com/electron/electron/blob/master/docs/api/auto-updater.md#autoupdatersetfeedurlurl-requestheaders)
`autoUpdateCheckOnStart`|whether to check for updates on app start
`rebuildNativeNodeModules`|turn on or off recompiling native modules, [more](#native-modules-support)
`webAppStartupTimeout`|amount of time after which the downloaded version is considered faulty if Meteor app did not start - [more](#hot-code-push-support)
`window`|production options for the main window - see [here](https://github.com/electron/electron/blob/master/docs/api/browser-window.md#new-browserwindowoptions)
`windowDev`|development options for the main window, applied on top of production options
`uglify`|whether to process the production build with uglify
`plugins`|meteor-desktop plugins list
`dependencies`|npm dependencies of your desktop app, the same like in `package.json`, only explicit versions are supported - check [here](#supported-dependency-version-types)
`packageJsonFields`|fields to add to the generated `package.json` in your desktop app
`builderOptions`|[`electron-builder`](https://github.com/electron-userland/electron-builder) [options](https://github.com/electron-userland/electron-builder/wiki/Options)
`packagerOptions`|[`electron-packager`](https://github.com/electron-userland/electron-packager) [options](https://github.com/electron-userland/electron-packager/blob/master/docs/api.md)

##### Applying different window options for different OS

You can use `_windows`, `_osx`, `_linux` properties to set additional settings for different OS. 
The default `settings.json` is already using that for setting a different icon for OSX.

##### Supported dependency version types

Only explicit versions are supported to avoid potential problems with different versions being 
installed. It is no different from `Meteor` because the same applies to adding `Cordova` plugins.

You can however use a local path to a npm package - and that will not be forbidden. But you need 
to keep track what has been distributed to your clients and what your current code is expecting 
when releasing a HCP update. 

### desktop.js

The `desktop.js` is the entrypoint of your desktop app. Let's take a look what references we 
receive in the constructor.
```javascript
    /**
     * @param {Object} log         - Winston logger instance
     * @param {Object} skeletonApp - reference to the skeleton app instance
     * @param {Object} appSettings - settings.json contents
     * @param {Object} eventsBus   - event emitter for listening or emitting events
     *                               shared across skeleton app and every module/plugin
     * @param {Object} modules     - references to all loaded modules
     * @param {Object} Module      - reference to the Module class
     * @constructor
     */
    constructor({ log, skeletonApp, appSettings, eventsBus, modules, Module })
```
Some of the references are describe in detail below:

#### `skeletonApp`

This is a reference to the Skeleton App. Currently there are only two methods you can call.  
`isProduction` - whether this is a production build  
`removeUncaughtExceptionListener` - removes the default handler so you can put your own in place

#### `eventsBus`

This is just an `EventEmitter` that is an event bus meant to be used across all entities running 
in the `Electron`'s main process (`.desktop`). Currently there are several events emitted on the 
bus by the `Skeleton App` that you may find useful:

event name|payload|description
----------|-------|------------
`unhandledException`| |emitted on any unhandled exceptions, by hooking to it you can run code before any other handler will be executed   
`beforePluginsLoad`| |emitted before plugins are loaded
`beforeModulesLoad`| |emitted before internal modules and modules from `.desktop` are loaded
`beforeDesktopJsLoad`| |emitted before `desktop.js` is loaded
`desktopLoaded`|`(desktop)`|emitted after loading `desktop.js`, carries the reference to class instance exported from it 
`afterInitialization`| |emitted after initialization of internal modules like HCP and local HTTP server
`startupFailed`| |emitted when the `Skeleton App` could not start you `Meteor` app  
`beforeLoadFinish`| |emitted when the `Meteor` app finished loading, but just before the window is shown  
`loadingFinished`| |emitted when the `Meteor` app finished loading (also after HCP reload)  
`windowCreated`|`(window)`|emitted when the [`BrowserWindow`](https://github.com/electron/electron/blob/master/docs/api/browser-window.md) (`Chrome` window with `Meteor` app) is  created, passes a reference to this window 
`newVersionReady`|`(version, desktopVersion)`|emitted when a new `Meteor` bundle was downloaded and is ready to be applied  
`revertVersionReady`|`(version)`|emitted just before the `Meteor` app version will be reverted (due to faulty version fallback mechanism) be applied  

Your can also emit events on this bus as well. A good practice is to namespace them using dots,
like for instance `myModule.initalized`.

#### `modules`

Object with references to other modules and plugins. Plugins can be found under their names i.e., 
`modules['meteor-desktop-splash-screen]`.  
Any module can be found under the name from `module.json`. 
Internal modules such as `autoupdate` and `localServer` are also there. You can also get reference to the `desktop.js` from `modules['desktop']` (note that the reference is also passed in 
the `desktopLoaded` event).

#### `Module`

Class that provides a way of defining API reachable by `Meteor` app - [more](#module---desktop-side).

## Writing modules

Module is just an encapsulated piece of code. Usually you would just provide certain type of 
grouped functionality in it. You can treat it like a plugin to your desktop app.  
One important rule is that you should not import files from the outside of your module directory 
as this will cause you problems when writing tests.  
You can always reach to other modules through `modules` and you can as well add a module with 
 some common code or utils.
Every module lives in its own directory and has to have a `module.json` file. Currently there are
 only four fields there supported:
- `name` - name of your module, will be used as a key in `modules` object
- `dependencies` - list of npm deps
- `extract` - list of files that should be excluded from packing into `.asar` (e.g. executables, 
files meant to be changed etc)
- `settings` - this object is passed as `settings` field in the object passed to module constructor 

#### `extract`
A little bit more about this. Files should be listed in a form of relative path to the module 
directory without any leading slashes, for example `extract: [ "dir/something.exe" ]` will be 
matched to `.desktop/modules/myModule/dir/something.exe`.

To path to your extracted files is added to your module `settings` as `extractedFilesPath`
. So your module constructor can look like this:
```javascript
import path from 'path';
constructor({ log, skeletonApp, appSettings, eventsBus, modules, settings, Module }) {
    this.pathToExe = path.join(settings.extractedFilesPath, 'dir/something.exe');
}
```

## Hot code push support

Applications produced by this integration are fully compatible with `Meteor`'s hot code push 
mechanism.  
The faulty version recovery is also in place - [more about it here](https://guide.meteor.com/mobile.html#recovering-from-faulty-versions). You can configure the timeout via 
`webAppStartupTimeout` field in `settings.json`.  

Versions are downloaded and served from [`userData`](https://github.com/electron/electron/blob/master/docs/api/app.md#appgetpathname) directory.
There you can find `autoupdate.json` and `versions` dir. If you want to return to first 
bundled version just delete them.

You can also analyze `autoupdate.log` if you are experiencing any issues.

## `Meteor.isDesktop`

In your `Meteor` app to run a part of the code only in the desktop context you can use `Meteor
.isDesktop`. Use it the same way you would use `Meteor.isClient` or `Meteor.isCordova`.
 
## `Desktop` and `Module`

### `Module` - desktop side
Use it to declare your API on the desktop side.
```javascript
    this.module = new Module('myModuleName');
```
[Documentation of the Module API](docs/api/module.md) - basically, it reflects [`ipcMain`]
(https://github.com/electron/electron/blob/master/docs/api/ipc-main.md).  

The only addition is the `respond` method which is a convenient method of sending response to 
`Desktop.fetch`. The `fetchId` is always the second argument received in `on`.  
Here is an [usage example](https://github.com/wojtkowiak/meteor-desktop-localstorage/blob/master/src/index.js#L31).
 

### `Desktop` - Meteor side
[Documentation of the Desktop API](docs/api/desktop.md) - reflects partially [`ipcRenderer`](https://github.com/electron/electron/blob/master/docs/api/ipc-renderer.md)<sup>*</sup>.    

<sup>* `sendSync` and `sendToHost` are not available</sup>

Use it to call and listen for events from the desktop side.

The only difference is that you always need to precede arguments with module name.
There are two extra methods:  
- **fetch** - like send but returns a `Promise` that resolves to a response
- **sendGlobal** - alias for `ipcRenderer.send` - if you need to send an IPC that is not namespaced

Example of `send` and `fetch` usage - [here](https://github.com/wojtkowiak/meteor-desktop-localstorage/blob/master/plugins/localstorage/localstorage.js#L9).  

## desktopHCP - `.desktop` hot code push
> #### experimental! 

There is an experimental support for hot code push of the `.desktop` directory.  
It works similarly to the `Meteor`'s builtin one. It also produces a `version` and 
`compatibilityVersion` to detect whether the update can be made.  
In `Meteor` whenever you change any of your `Cordova` dependencies (add/remove/change version) 
you will make an incompatible change meaning that a new version will not be hot code pushed.  
The same applies here. In this case your desktop dependencies are npm packages.   
To make it clear, **npm packages are not hot code pushed** - only contents of `.desktop` are.

The `compatibilityVersion` is calculated from combined list of:
 - dependencies from `settings.json`
 - plugins from `settings.json`
 - dependencies from all modules in `.desktop/modules`
 - major and minor version of `meteor-desktop` (X.X.Y - only X.X is taken)
 - major version from `settings.json` (X.Y.Y - only X is taken).
 
Until 1.0 major and minor version of `meteor-desktop` will be used. From 1.0, semver will be 
followed and only major version change will mean that a version is backwards incompatible. 

#### How this works
Two Meteor plugins are added to your project - bundler and watcher. Bundler prepares the `desktop.asar` which is then added to you project as an asset.   
Watcher just watches for file changes and triggers project rebuilds.   

#### Caveats 
- desktop app needs to be restarted when a new bundle is applied
- the bundled desktop app goes over normal HCP mechanism meaning that a `desktop.asar` file will 
also be 
distributed to your mobile clients and cause unnecessary updates in case you only made changes in
 `.desktop`
- if errors in your app prevented startup during development, change in `.desktop` will not 
trigger project rebuild
- if errors in `.desktop` prevented startup, watcher will not work and you need to make any 
change in `version` field in the `desktop.version` to trigger rebuild (this file is in the root of 
your project)
- if your run a production build of your desktop app it will not receive updates from project run
 from `meteor` command unless you run it with `--production` - that is because development build 
 has `devtron` added and therefore the `compatibilityVersion` is different  
- after reload logs will no longer be shown in the console

## How to write plugins

Plugin is basically a module exported to a npm package. `module.json` is not needed and not taken
 into account because `name` and `dependencies` are already in `package.json`. Also you can not use 
 the `extract` functionality as that only works in modules. Plugin `settings` are set and taken 
 from the `plugins` section of `settings.json`. [Here](scaffold/settings.json#L26) is an example of passing settings to splash 
 screen plugin.  
#### `meteorDependencies` in `package.json` 
One extra feature is that you can also depend on Meteor packages through `meteorDependencies` 
field in `package.json`. Check out [`meteor-desktop-localstorage`](https://github.com/wojtkowiak/meteor-desktop-localstorage/blob/master/package.json#L52) for example.  
A good practice when your plugin contains a meteor plugin is to publish both at the same version. 
You can then use `@version` in the `meteorDependecies` to indicate that the Meteor plugin's 
version should be equal to npm package version.

If you made a plugin, please let us know so that it can be listed here.
##### List of known plugins:
[`meteor-desktop-splashscreen`](https://github.com/wojtkowiak/meteor-desktop-splash-screen)  
[`meteor-desktop-localstorage`](https://github.com/wojtkowiak/meteor-desktop-localstorage)

## Squirrel autoupdate support

Squirrel Window and OSX autoupdates are supported. So far the only tested server is 
[`electron-release-server`](https://github.com/ArekSredzki/electron-release-server) and the 
default url `http://127.0.0.1/update/:platform/:version` provided in `settings.json` assumes you 
will be using it.  
The `:platform` and `:version` tags are automatically replaced by correct values.   
You can hook into Squirrel Windows events in `squirrelEvents.js` in `.desktop`.

More:  
https://github.com/electron/electron/blob/master/docs/api/auto-updater.md  
https://github.com/ArekSredzki/electron-release-server

## Native modules support

This integration fully supports rebuilding native modules (npm packages with native node modules)
 against `Electron`'s `node` version. However, to speed up build time, it is **switched off by default**. 

If you have any of those in your dependencies, or you know that one of the packages or plugins is using it, you should turn it on by setting `rebuildNativeNodeModules` to true in your `settings.json`. Currently there is no mechanism present that detects whether the rebuild should be run so it is fired on every build. A cleverer approach is planned before `1.0`. 

## Devtron

[`Devtron`](http://electron.atom.io/devtron/) is installed and activated by default. It is 
automatically removed when building with `--production`. As the communication between your Meteor
 app and the desktop side goes through IPC, this tool can be very handy because it can sniff on 
 IPC messages.
<kbd>![devtron IPC sniff](docs/devtron_ipc.gif)</kbd>

## Testing desktop app and modules

For unit tests you should not have problems with using [electron-mocha](https://github.com/jprichardson/electron-mocha).  
For functional testing [Spectron](http://electron.atom.io/spectron) should be used.

There are two exemplary tests present in the default scaffold. Check them out as they have some 
comments in them.  
To run them you need to init functional test support by invoking:
```
npm run desktop -- init-tests-support
```
Two tasks should be added to your `scripts` section: `test-desktop` and `test-desktop-watch`.
Feel free to run the tests with: `npm run test-desktop`.

For testing modules there is a [test suite](https://github.com/wojtkowiak/meteor-desktop-test-suite) available.
It is used extensively in the plugins (splash screen & localstorage) tests so you can check there 
for more examples. 

## `MD_LOG_LEVEL`
`MD_LOG_LEVEL` env var is used to set the logger verbosity. Currently before `1.0` it is set to 
`ALL` but you can change it to any of `INFO, WARN, ERROR, DEBUG, VERBOSE, TRACE`. You can also 
select multiple levels joining them with a comma, for example: `INFO,WARN`.

## Packaging

`npm run desktop -- package <ddp-url>`

This produces a package using [`electron-packager`](https://github.com/electron-userland/electron-packager).  
Package is produced and saved in `.desktop-package` directory. You can pass options via `packagerOptions` in 
`settings.json`.

## Building installer

`npm run desktop -- build-installer <ddp-url>`

This packages and builds installer using [`electron-builder`](https://github.com/electron-userland/electron-builder).  
Installer is produced and saved in `.desktop-installer` directory. You can pass options via 
`builderOptions` in `settings.json`.

Please note that `electron-builder` does not use `electron-packager` to create a package. So the 
options from `packagerOptions` are not taken into account.


## Roadmap 
> <sup>aka road to 1.0</sup>

This version should be considered as beta version.  
Any feedback/feature requests/PR is highly welcomed and highly anticipated.  
There is rough plan to publish a release candidate around January 2017. Until that expect things to 
change rapidly and many frequent 0.X.X releases.
You can find the roadmap filtering by milestone and `accepted` tag on the github issues list [here](https://github.com/wojtkowiak/meteor-desktop/issues?q=is%3Aissue+is%3Aopen+label%3Aaccepted).  

## Contribution

PRs are always welcome and encouraged. If you need help at any stage of preparing a PR, just 
file an issue. It is also good, to file a feature request issue before you start working to 
discuss the need and implementation approach.

Currently this package does not work when linked with `npm link`. To set up your dev environment 
it is best to create a clean `Meteor` project, add `meteor-desktop` to dependencies with a relative
 path to the place where you have cloned this repo and in scripts add `desktop` with `node
 ./path/to/meteor-desktop/dist/bin/cli.js`.  
 Also to make changes in the desktop HCP plugins run `Meteor` project with `METEOR_PACKAGE_DIRS` 
 set to `/absolute/path/to/meteor-desktop/plugins` so that they will be taken from the cloned repo. 
  

## FAQ

## Changelog

- **0.1.1** - `meteor-desktop-splash-screen` version in the default scaffold updated to [`0.0.31`](https://github.com/wojtkowiak/meteor-desktop-splash-screen#changelog) 
- **0.1.0** - first public release
