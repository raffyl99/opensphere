# Developing

## Setup

Start by [forking](https://github.com/openlayers/openlayers/fork) the OpenSphere repository.

### Prerequisites

* git
* Java 1.7.0+
* Node 6+ and npm
* Python (optional)

Ensure that the executables `git`, `node`, and `java` are in your `PATH`.

Install the dependencies by running

    $ npm install

### Linking

If you are working on several plugins and config projects, you may end up with a workspace like

    workspace/
        opensphere/
        opensphere-config-developer/
        some-common-lib/
        opensphere-plugin-x/
        opensphere-plugin-y/

`npm link` is designed to help with this, but can get cumbersome to maintain manually with many projects. We recommend [npm-workspace](https://www.npmjs.com/package/npm-workspace) to automate the links.

### Serving the application

While not required, we highly recommend setting up nginx or Apache httpd to server up your workspace from your local machine. This will allow you to easily proxy remote servers and mock up services to develop against in addition to serving the application exactly as it will be served from production rather than accessing it via a file path in the browser or serving it with a node-based server.

## The Build

OpenSphere has all of its build targets as npm scripts. Therefore you can run any particular target by running:

    $ npm run <target>

The most common targets are

    $ npm run build       # runs the full build for both debug and compiled mode
    $ npm run build:debug # runs the debug build only (requires Python) 
    $ npm run test        # runs the unit tests
    $ npm run test:debug  # runs the unit tests with a configuration more suited to debugging
    $ npm run apidoc      # generates api documentation
    
Each target runs its individual pieces through npm scripts as well. Several of those pieces are highly useful when run by themselves just to see if you fixed an error in that part of the build before restarting the entire thing.

    $ npm run lint             # runs the linter to check code style
    $ npm run compile:resolve  # runs the resolver to check dependency/plugin/config resolution
    # npm run compile:gcc      # runs the google-closure-compiler to produce the compiled JS
    # npm run compile:css      # runs node-sass to produce the minified/combined css

### The resolver

[opensphere-build-resolver](https://github.com/ngageoint/opensphere-build-resolver) runs through all of an applications dependencies, plugins (opensphere-plugin-x), or config projects (opensphere-config-y) and then the resolver's plugins produce arguments for the compiler, arguments for node-sass, page templates for conversion, and more! All of these files are written to the `.build` directory and used later in the build.

### The Google Closure Compiler

Use of the [Closure Compiler](https://developers.google.com/closure/compiler/) has been limited among the open source community. However, unlike other projects which produce minified Javascript, the Closure Compiler is a true compiler. It does type checking, optimizations, and dead code removal. Type checking is essential to any large project, and the other optimizations allow our compiled code (in some cases) to perform three times better than our unminified code.

We use the compiler's `ADVANCED` compilation level, which is [described in detail here](https://developers.google.com/closure/compiler/docs/api-tutorial3). Also check out the [annotations](https://developers.google.com/closure/compiler/docs/js-for-compiler) available for the compiler.

Because the Closure Compiler does so much more than just minification, the build takes a non-trivial amount of time to run. To help with developer productivity, we have produced a build system which does not need to be rerun when files change. Instead, it only needs to be run when files are added or dependencies change.

Some of the intricacies from using the compiler are documented in the [Compiler Caveats](#caveats) section below.

### The debug build output

The `index-template.html` and its corresponding `index.js` file define how the main page is packaged up by [opensphere-build-index](https://github.com/ngageoint/opensphere-build-index). That script produces `index.html`, which is the is the debug instance. It contains all of the vendor scripts and css in addition to all of the source files listed from the Closure Compiler manifest (`.build/gcc-manfiest`).

If you set up nginx or httpd as recommended above, accessing it might be accomplished by pointing your browser at

    http://localhost:8080/workspace/opensphere

Note: because the debug instance references each individual Javascript file in place, it can result in the debug page referencing thousands of individual files. The only browser that handles this gracefully (as of this writing) is Chrome. Firefox technically works but is much more painful.

Once you have run the build once, you can make changes to files in the workspace and pick them up on the page by merely refreshing it. The build only has to be run if dependencies (`goog.require/provide`) change or if files are added or removed.

### The compiled build output

The compiled build output is available in `dist/opensphere`. You will need to test your changes in both places, but generally compiled mode should be checked after you have largely completed the feature on which you are working. It does contain source maps for debugging, and also loads much quicker in Firefox and IE since all the code is compiled and minified to a single file.

## Testing

All of our unit tests for opensphere are written in [Jasmine](https://jasmine.github.io/) and run with [karma](https://karma-runner.github.io/1.0/index.html) via `npm test`. Detailed coverage reports are available in `.build/test/coverage`. If you are writing a plugin or standalone application, you are free to use whatever testing framework you like, but you'll get more for free if you use what we've set up for you already. If you want to switch out Jasmine with something else (or a newer version of Jasmine), that should also be doable.

Any contributions to OpenSphere should avoid breaking current tests and should include new tests that fully cover the changed areas.

## Developing plugins

See our [plugin guide](https://github.com/ngageoint/opensphere/blob/master/PLUGIN_DEVELOPMENT.md) to get started developing plugins.

## Using OpenSphere as a library

See our [application guide](https://github.com/ngageoint/opensphere/blob/master/APP_DEVELOPMENT.md) to get started using OpenSphere as a library for your own application.

## <a name="caveats"></a> Compiler Caveats

The compiler will attempt to minify/rename everything not in a string. For the most part, this is fine. However, when working with Angular templates, the variable/function names used in the template itself will not be replaced. To combat this, we use bracket notation for variables such as `$scope['value'] = 0`, and we use `goog.exportProperty()` on controller methods that should be made available to the UI.

Broken Example:

    /**
     * @param {!angular.Scope} $scope The scope
     */
    package.DirCtrl = function($scope, $element) {
      $scope.value = 3;
    };

    /**
     * @param {number} value
     */
    package.DirCtrl.prototype.isPositive = function(value) {
      return value > 0;
    };

    // template
    <span ng-show="ctrl.isPositive(value)">{{value}} is positive</span>

This will work great in debug mode (no minification), but will fail in compiled mode. To fix this, we need to ensure that the compiled build does not minify the two items we used in the template.

Fixed Example:

    /**
     * @param {!angular.Scope} $scope The scope
     */
    package.DirCtrl = function($scope, $element) {
      $scope['value'] = 3;
    };

    /**
     * @param {number} value
     */
    package.DirCtrl.prototype.isPositive = function(value) {
      return value > 0;
    };
    // we highly recommend making this a snippet
    goog.exportProperty(package.DirCtrl.prototype, 'isPositive', package.DirCtrl.prototype.isPositive);

    // template
    <span ng-show="ctrl.isPositive(value)">{{value}} is positive</span>

Now it works in compiled mode! Note that UI templates is not the only place were bracket notation is useful. It is useful wherever you want to have the compiler skip minification.
