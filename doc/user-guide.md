# Stripes CLI User Guide

Note: When serving or building an existing app module that has dependencies on unreleased versions of other Stripes modules, be sure to use the `npm-folioci` registry.  This applies whether you've installed the CLI from `npm-folio` or `npm-folioci`.

* [Using the CLI](#using-the-cli)
    * [Installation](#installation)
    * [Options](#options)
    * [Standard input](#standard-input)
    * [Help](#help)
    * [Sub-commands](#sub-commands)
    * [Interactive commands](#interactive-commands)
* [Configuration](#configuration)
    * [Module export](#module-export)
    * [Environment variables](#environment-variables)
* [Background](#background)
    * [CLI Context](#cli-context)
    * [Platforms](#platforms)
    * [Aliases](#aliases)
* [Development prerequisites](#development-prerequisites)
* [App development](#app-development)
    * [Creating your app](#creating-your-app)
    * [Assigning permissions](#assigning-permissions)
    * [Running your app](#running-your-app)
    * [Running tests](#running-tests)
    * [Including another Stripes module](#including-another-stripes-module)
* [Platform development](#platform-development)
    * [Environment setup](#environment-setup)
    * [Creating a platform](#creating-a-platform)
    * [Running a platform](#running-a-platform)
    * [Running tests for a platform](#running-tests-for-a-platform)
    * [Updating the platform](#updating-the-platform)
* [Interacting with Okapi](#interacting-with-okapi)
    * [Managing UI modules](#managing-ui-modules)
    * [Managing UI permissions](#managing-ui-permissions)
* [Generating a production build](#generating-a-production-build)
    * [Analyzing bundle output](#analyzing-bundle-output)
    * [Reducing build output](#reducing-build-output)
* [Viewing diagnostic output](#viewing-diagnostic-output)
    * [Observing Okapi requests](#observing-okapi-requests)

## Using the CLI

Stripes CLI is invoked with the `stripes` command.  When Stripes CLI is [installed](../README.md#installation) globally, Yarn will make the `stripes` command available in your path.  To run a given command, run `stripes` followed by the desired command name.

Example:
```
stripes serve
```

### Installation

See the [README](../README.md#installation) for installation and upgrade instructions.

### Options

Any option can be passed to the CLI either on the command line, as an [environment variable](#environment-variables), or in a `.stripesclirc` [configuration file](#configuration).

Options passed on the command line are prefixed with `--` in the form of `--optionName value`.

```
stripes serve --port 8080
```

Notes:
* Boolean options are considered true by simply passing the option name.  No value is required.
* To explicitly set a Boolean option to false, prefix the option name with `--no-` as in `--no-optionName`.
* String options can be wrapped in quotes when spaces are desired. This is helpful for descriptions.
* Array option values are space-separated.

Example passing array values for `modules` and false for `install`:
```
stripes workspace --modules ui-users stripes-core stripes-components --no-install
```

### Standard input

Some commands support passing an option via standard input (stdin).  Where supported, `stdin` is typically accepted as a whitespace-delimited (including line breaks) list of values.

For example, a file containing the module descriptor ids, one per line, can be piped to a command using `stdin`.

File: `my-modules`
```
folio_users-2.12.2
folio_inventory-1.0.0
folio_checkout-1.1.0
folio_checkin-1.1.0
```

The above file can be piped to `mod enable` to enable multiple module descriptor ids for a tenant:
```
cat my-modules | stripes mod enable --tenant diku
```

Various commands produce output that is suitable for piping to related commands.  For example, `perm view` will output a list of permission names, one on each line.  This can be optionally be filtered and piped to `perm assign`.

```
stripes perm view --user jack | grep hello-world | stripes perm assign --user jill
```

Support for `stdin` is indicated in a command option's notes in the [command reference](./commands.md).

Option | Description | Type | Notes
---|---|---|---
`--ids` | Module descriptor ids | array | *supports stdin*


### Help
Every command in Stripes CLI includes a description, list of options with descriptions, and often example usages.  To view help for any command, simply pass the `--help` option to the command.

```
stripes serve --help
```

### Sub-commands
Related CLI commands are often grouped together.  This for organizational purposes.

Example "sub-commands" of the `platform` command:
```
stripes platform clean
stripes platform pull
```

### Interactive commands
Some commands may require additional input before continuing.  When necessary, a command may prompt the user with questions.  This interactive input can be disabled entirely by passing `--no-interactive`.  This is useful when the CLI is part of an automated script.

```
stripes app create "Hello World" --no-interactive
```

Note: This will force default values, if available, to be used.  When no suitable defaults are available, the command may fail or produce unexpected results.  Please verify behavior first.


## Configuration
Frequently used options can be saved to a `.stripesclirc` configuration file to avoid entering them each time.  Stripes CLI will use the configuration file found in the current working directory, or the first one found walking up the tree.  The default configuration file format is JSON.

Any supported command-line positional or option can be defined.  For example:
```json
{
  "configFile": "stripes.config.js",
  "port": 8080
}
```

In addition to command-line options, aliases for Stripes UI modules are also supported.  Aliases paths should be relative to the directory containing the `.stripesclirc` config file which defines the aliases.
```json
{
  "aliases": {
    "@folio/users": "../ui-users"
  }
}
```

### Module export
In addition to JSON, the CLI configuration may be authored as a JavaScript module export.  This is useful for generating options dynamically or defining [CLI plugins](./dev-guide.md#plugins). When defining a JavaScript module export, be sure to use the `.js` file extension.

Example `.stripesclirc.js`:
```javascript
const environment = process.env.NODE_ENV;
let url;

if (environment === 'sandbox') {
  url = 'https://okapi-sandbox.frontside.io';
} else {
  url = 'https://okapi.frontside.io';
}

module.exports = {
  okapi: url,
  tenant: 'fs',
  install: true,
}
```

### Environment variables
Any CLI option can be set using environment variables prefixed with `STRIPES_`.  For example, to specify the `--port` option, use an environment variable named `STRIPES_PORT`.


## Background

### CLI Context

CLI operations may vary depending on the context in which the command is run.  By identifying a context, the CLI can validate the command is appropriate and, in some cases, modify workflow as needed.  Context is determined by the `package.json` in the working directory.  Use the `status` command to view the current context.

Types:
* `APP` - Identified by the value of the `stripes.type` property in `package.json`.  The CLI will automatically generate a virtual platform when serving an UI app module in isolation.
* `PLATFORM` - Identified by a package.json containing one or more `@folio/` dependencies, but no `stripes` object.
* `WORKSPACE` - CLI is run from a directory containing a Yarn workspace package.json.
* `EMPTY` - No `package.json` detected.  Suitable for creating new UI apps or platforms.
* `CLI` - Command is run from the Stripes CLI directory.


### Platforms

Stripes UI modules are meant to be built together with other modules in a platform that shares common build infrastructure.  A platform consists of a `package.json` and a tenant configuration typically named `stripes.config.js`. See the [Stripes Sample Platform](https://github.com/folio-org/stripes-sample-platform) for a good example.

The platform that Stripes CLI uses is influenced by the CLI context and constructed in the following order:

1. Base configuration:  When a file argument like `stripes.config.js` is provided, this will be used as the base.  Otherwise, the CLI will use its own internal defaults that contain no modules.

2. Virtual configuration:  In the APP context, the CLI will apply the current app as a module and generate an alias for the app to be run in isolation.  In the PLATFORM or APP context, the CLI will then add modules for all aliases defined, but only when an explicit module configuration is absent from `stripes.config.js`, or no `stripes.config.js` has been provided.

3. Command configuration: Any relevant options passed in on the command line are applied to the configuration last.

Tip: Use the `status` command (optionally with a file and/or other config options) to view the CLI's generated platform configuration in the current context.



### Aliases

Aliases are used to associate platform modules with local code repositories in a development environment.  An alias is comprised of a key-value pair where the key is a module (`@folio/users`) and the value is the relative path to where the module can be found (`../ui-users`).  Aliases are not limited to `@folio` scope modules.  At build time, any defined aliases will be applied to the Webpack configuration as Webpack aliases.

The initial goals of defining aliases was to facilitate serving an app in isolation, as well as eliminate the need for Yarn-linking modules for platform development.  The latter can be mitigated by using Yarn workspaces for the platform developer, however, aliases can still provide some advantages.  Aliases are easily added ad-hoc for debugging or testing.  They can work with modules outside the workspace, or used to share across workspaces.

There are two methods of adding aliases in the CLI:

1) `alias` command - This command manages aliases in a global collection.  Aliases defined with the `alias` command will be available for all commands, regardless of where they are run, until the alias is removed.  This command is useful for adding aliases ad-hoc and sharing an alias across multiple apps, platforms, or workspaces. See the [`alias` command](./commands.md#alias-command) for more detail.

1) `.stripesclirc` file - Any aliases defined in a CLI configuration file apply to commands run from the directory containing `.stripesclirc` file.  Use the configuration file when adding aliases in bulk or looking for a consistent set of alias. See the [CLI configuration](#configuration) for more information.

## Development prerequisites

The subsequent development sections assume an existing Okapi backend, such as the [FOLIO testing-backend](https://app.vagrantup.com/folio/boxes/testing-backend) Vagrant box, is installed and running locally for front-end development.  With Vagrant installed, you can set up FOLIO testing-backend with the following commands:
```
$ mkdir testing-backend
$ cd testing-backend
$ vagrant init folio/testing-backend
$ vagrant up
```
The FOLIO testing-backend vagrant box includes the default tenant, "diku".  See [this stripes document](https://github.com/folio-org/stripes/blob/master/doc/new-development-setup.md#update-your-vm) for information updating your Vagrant box.

As an alternative to using a local Vagrant box for Okapi, you can develop against an external Okapi instance.  To do this, specify the URL of your Okapi using the `--okapi` option with each command.  This can also be passed via a `.stripesclirc` [configuration file](#configuration).

```json
{
  "okapi": "http://your-okapi-host:and-port",
  "tenant": "your-tenant-id"
}
```

## App development

As a UI app developer, it is often preferred to develop your app independent from an entire platform.  This allows you to focus on your app's own code and not worry about how it is built or integrated within FOLIO.  The Stripes CLI provides all the necessary configuration to develop both new and existing apps in isolation.

Prerequisites:  An Okapi backend is required. See [development prerequisites](#development-prerequisites)

### Creating your app

From a suitable directory, run the following:
```
stripes app create "Hello World" --install
```

This generates a skeleton Stripes UI app with sample routes and settings.  The CLI will transform the provided app name, "Hello World", to follow naming conventions where the `ui-` prefix or `@folio` scope is used.

```
Creating app...
{
  "appName": "hello-world",
  "appDescription": "an example app",
  "appDir": "ui-hello-world",
  "uiAppName": "ui-hello-world",
  "packageName": "@folio/hello-world",
  "displayName": "Hello World",
  "appRoute": "/helloworld",
  "componentName": "HelloWorld"
}
```

The `--install` option prompts the CLI to automatically run `yarn install` on the directory afterwards.  If the install option was omitted, please `cd` to the app's directory and run `yarn install`.

From here you can immediately start [running your app](#running-your-app), but it is best to properly post the app's module descriptor to Okapi and [assign permissions](#assigning-permissions).

*Tip:* If you've already [logged into Okapi](#interacting-with-okapi), you can do this all with one command:

```
stripes app create "Hello World" --assign diku_admin
```

In the above command, after creating an app and installing dependencies, `--assign` will first post the module descriptor to Okapi and then enable the module for the tenant.  Next, it will assign the new app's default permissions to the user `diku_admin`.  See [assigning permissions](#assigning-permissions) below for details on how to perform these operations independently, or to add permissions later on during development.


### Assigning permissions

The new app created above contains the following permissions sets that Okapi needs to know about via module descriptor.  Once Okapi has the new app's module descriptor, the app can be assigned to a tenant, and permissions assigned to a user.  Doing so eliminates the need for setting `--hasAllPerms` during development.

```
"permissionSets": [
  {
    "permissionName": "module.hello-world.enabled",
    "displayName": "UI: Hello World module is enabled",
    "visible": true
  },
  {
    "permissionName": "settings.hello-world.enabled",
    "displayName": "Settings (hello-world): display list of settings pages",
    "subPermissions": [
      "settings.enabled"
    ],
    "visible": true
  }
]
```

To push your app's module descriptor, use the `mod add` command from within the app's directory.
```
stripes mod add
```

Next enable the module descriptor for your tenant:
```
stripes mod enable --tenant diku
```

Finally assign the app's default enabled permissions for a user:
```
stripes perm assign --name module.hello-world.enabled --user diku_admin
stripes perm assign --name settings.hello-world.enabled --user diku_admin
```

See Stripes-core's [Adding Permissions](https://github.com/folio-org/stripes-core/blob/master/doc/adding-permissions.md) for more detail and on how to manually add permissions.


### Running your app

After creating "Hello World" and installing dependencies, the new app is ready to run.  Change to the new app directory and serve your new app using a development server:

```
stripes serve
```

To specify your own tenant ID or to use an Okapi instance other than the default http://localhost:9130, pass the `--okapi` and `--tenant` options or set them in `.stripesclirc` file.
```
stripes serve --okapi http://my-okapi.example.com:9130 --tenant my-tenant-id
```

Note: When serving up a newly created app that either does not have a module descriptor in Okapi, or permissions assigned to the user, pass the `--hasAllPerms` option to display the app in the UI navigation.  While handy for initial development, `--hasAllPerms` should not be used in production builds. See [assigning permissions](#assigning-permissions) to eliminate the need for this.

```
stripes serve --hasAllPerms
```

### Running tests

The newly created app has some basic UI end-to-end tests included, designed to run with the Nightmare framework.  To run these tests, use the `test nightmare` command:

```
stripes test nightmare --run demo --show
```

The `--run` option specifies the tests, in this case the sample tests included with our app are named "demo".  The `--show` option will display the UI while running the tests.

Additional options specific to the `@folio/ui-testing` framework can be passed in the form of `--uiTest.optionName value`.  For example, to override the default `typeInterval`, pass `--uiTest.typeInterval 50`.  Refer to [@folio/ui-testing](https://github.com/folio-org/ui-testing) for more information on available options.


### Including another Stripes module

Now that our Hello World app is up and running on its own, we may want to bring in an existing app for testing or further development.  The CLI makes this easy.  The following will demonstrate how to add `ui-users`.

From the directory above `ui-hello-world`, clone `ui-users` and install its dependencies.
```
git clone https://github.com/folio-org/ui-users.git
cd ui-users
yarn install
```

We should now have the following directory structure:
```
myDir
├─ui-hello-world
└─ui-users
```

Next add an alias for ui-users. Provide a relative path to the ui-users directory. Given you're in /ui-hello-world:
```
stripes alias add @folio/users ../ui-users
```

Now simply start the app up again.  From the ui-hello-world directory, run:
```
stripes serve
```

The FOLIO platform generated will now include ui-users!  The same procedure can be followed to include non-app modules as well such as `stripes-components` and `stripes-core`.

Note: When adding an alias via the `alias add` command, the alias is considered global and will remain in effect for any command, run from any directory, until removed with `alias remove`.


## Platform development

When developing multiple Stripes apps and/or core modules at the same time, it is often desired to work with a platform containing most or all of the available FOLIO Stripes modules.  See [new development setup](https://github.com/folio-org/stripes/blob/master/doc/new-development-setup.md) in stripes-core for more details.  The Stripes CLI provides commands to simplify the creation of such platforms, consolidating several of the steps.

Prerequisites:  An Okapi backend is required. See [development prerequisites](#development-prerequisites)

### Environment setup

The easiest way to manage all the apps within a platform is with a [Yarn workspace](https://yarnpkg.com/lang/en/docs/workspaces/).  The CLI's `workspace` command will help set up a workspace-based development environment that is ready to go.

From a suitable directory, run the following:
```
stripes workspace
```

After prompting for modules, the `workspace` command will generate a directory named "stripes", clone all the selected modules, and install their dependencies.  For any platforms chosen during module selection, such as `stripes-sample-platform`, a local Stripes configuration (`stripes.config.js.local`) will be generated.

If a directory other than "stripes" is desired, use the `--dir` option.
```
stripes workspace --dir temp
```

To skip the interactive module selection and just include everything, pass `all` to the `--modules` option.
```
stripes workspace --modules all
```

*Note about workspaces:* Although similar in that both "workspace" and "platform" may be used to mean "development environment", they are both independent things.  A Yarn workspace is a method for managing a dependencies across multiple modules, whereas a FOLIO platform defines a collection of Stripes modules configured for FOLIO.  One workspace may contain multiple platforms.  Also, a platform can exist without a Yarn workspace.


### Creating a platform

If you selected a platform such as `stripes-sample-platform` or `platform-core` at the time of creating a workspace, you are all set.  Refer to `stripes.config.js` and `package.json` of the [Stripes Sample Platform](https://github.com/folio-org/stripes-sample-platform) if you wish to create a platform manually. 


### Running a platform

After creating a workspace with a selected platform, you can immediately serve it up.  Change directories to your platform and run `stripes serve` with a tenant configuration file, such as `stripes.config.js` or `stripes.config.js.local`.

```
cd stripes/stripes-sample-platform
stripes serve stripes.config.js.local
```


### Running tests for a platform

See the [ui-testing readme](https://github.com/folio-org/ui-testing/blob/master/README.md).

TODO: Document CLI-specific operations that help with this.


### Updating the platform

To pull the latest code changes from master for all cloned modules in your platform, run `platform pull` from either the platform directory, or the Yarn workspace directory.

```
stripes platform pull
```

When run from a platform directory, the CLI will pull the latest code for all aliased modules in the platform wherever they exist on the file system.  When run from a workspace directory, the CLI will pull the latest code for all known Stripes apps/modules in the workspace directory.


## Interacting with Okapi

Some CLI commands make requests to a running Okapi instance.  For these to work, you must have a valid token.  To obtain a token, use the `okapi` command to login:

```
stripes okapi login diku_admin --okapi http://localhost:9130 --tenant diku
```

*Note:* When working with Okapi, it is easiest to set `okapi` and `tenant` options in a `.stripesclirc` file or environment variables.  Either method avoids the need to manually supply `--okapi` and `--tenant` with each command.  The remaining examples in this section assume `okapi` and `tenant` are already set via config file or environment variable.

```
stripes okapi login diku_admin
```

If not supplied after the username, the CLI will prompt for a password.  After a successful login, the token will be stored for use with future CLI commands.

To clear a previously saved token run:
```
stripes okapi logout
```

### Managing UI modules

The CLI `mod` command can be used to manage Stripes UI modules for a tenant.  All `mod` commands can operate on the app context of the current directory.  Therefore, be sure to `cd` into the desired app directory when working with a single app. The `mod enable` and `mod disable` commands have recently added support for [stdin](#standard-input) and can work with multiple modules descriptor ids.

To view the current app's module descriptor:
```
stripes mod descriptor --full
```

To add a module to Okapi and enable it for a tenant:
```
stripes mod add
stripes mod enable --tenant diku
```

To enable multiple modules for a tenant given an existing file, `my-modules`, containing module descriptor ids:
```
cat my-modules | stripes mod enable --tenant diku
```

To remove a module from Okapi:
```
stripes mod remove
```

When a module is already associated with a tenant it cannot be removed without disabling it for the tenant first.
```
stripes mod disable --tenant diku
```

When updating a module descriptor in Okapi, the CLI provides a shortcut.  The following will attempt to remove a module descriptor, and if necessary, disable it.  After adding the new module descriptor, the module will be re-enabled for the tenant.
```
stripes mod update
```
Note: At this time, `mod update` will only attempt to disable/enable one tenant (typical development use-case).  Multiple tenants will have to be disabled manually.


### Managing UI permissions

Create new UI permissions with the following command:
```
stripes perm create ui-hello-world.example --push --assign diku_admin
```

This will update the current app's `package.json` with a new permission and invoke the `mod add` or `mod update` command as needed to push the new module descriptor to Okapi.  Finally, the permission will be assigned to the username provided.

If the `--push` and `--assign` options are omitted (or the permissions were created manually in package.json), run the following commands to update Okapi and assign permissions to a user:

```
stripes mod update
stripes app perms | stripes perm assign --user diku_admin
```


## Generating a production build

To generate a build for production, it is best to use a clean platform install with no workspace or aliases defined.  The following describes how to build `platform-core`:

```
git clone https://github.com/folio-org/platform-core.git
cd platform-core
yarn install
stripes build stripes.config.js my-build-output
```

If source maps are desired, include them with the `--sourcemap` option.
```
stripes build stripes.config.js my-build-output --sourcemap
```

The generated build assets will be placed in the directory path provided (in this case, `my-build-output`).  These files are ready to serve up from the file server of your choice.  For testing purposes, you can serve up an existing Stripes build using the following command:
```
stripes serve --existing-build my-build-output
```

### Analyzing bundle output

Sometimes it is useful to visualize the contents of the build to identify areas that could use some optimization.  Stripes CLI includes the Webpack Bundle Analyzer to help with this.  To enable the bundle analyzer, pass the `--analyze` option to the `build` command:

```
stripes build stripes.config.js my-build-output --analyze
```

Note: If running the analyzer with aliased modules, duplication is likely.  It is best to run the analyzer on a platform-only install.


### Reducing build output

One quick way to limit the build output, is to limit the number of languages included in the build.  This is done my modifying a tenant's Stripes configuration.  See [filtering translations at build time](https://github.com/folio-org/stripes/blob/master/doc/dev-guide.md#filtering-translations-at-build-time) of the Stripes developer guide on how to to this.  The result will not only limit translation files, but also locale assets for `react-intl` and `moment` libraries.

Filtering languages can be done with the CLI by specifying the `languages` option which accepts an array of values.  
```
stripes build stripes.config.js --languages en es
```

## Viewing diagnostic output

The CLI's `status` command is a helpful starting point for diagnosing errors.  Among other things, `status` will output which stripes-core is in use, the stripes config for the current context and list any aliases. 

```
stripes status stripes.config.js
```

To view stripes-cli and stripes-core diagnostic output for any command while it is running, set the `DEBUG` environment variable to `stripes*`.  For example:
```
DEBUG=stripes* stripes serve stripes.config.js
```

On Windows set the environment variable using the `set` command:
```
set DEBUG=stripes*
```

See the [debugging section](./dev-guide.md#debugging) in the CLI developer guide for more information on `DEBUG` usage.


### Observing Okapi requests

Monitoring Okapi requests for any Stripes CLI command is easy. Set the DEBUG environment variable to `stripes-cli:okapi`.  This can be done by prefixing any CLI command with `DEBUG=stripes-cli:okapi`.  For example:

```
DEBUG=stripes-cli:okapi stripes okapi login diku_admin
```

This will show what Okapi endpoints were called during a CLI command and how Okapi responded.
```
  stripes-cli:okapi ---> POST http://localhost:9130/authn/login +0ms
  stripes-cli:okapi <--- 201 Created +136ms
User diku_admin logged into tenant diku on Okapi http://localhost:9130
```

To avoid having to prefix every command, simply set the DEBUG environment variable accordingly.
```
export DEBUG=stripes-cli:okapi
```

On Windows:
```
set DEBUG=stripes-cli:okapi
```
