# buildpack: R

This buildpack downloads and installs R into a Scalingo app image.

## Using this buildpack

The following instructions should get you started:

1. Initialize a new git repository wherever you want:

```bash
% mkdir my-r-app
% cd my-r-app
% git init
```

2. Create the Scalingo app:

```bash
% scalingo create my-r-app
```

3. [Add some R packages](#installing-r-packages), if needed.

4. Deploy:

```bash
% git push scalingo master
```

5. Start a `console` one-off:

(This will only work if you have a `console` process type defined in your
`Procfile`.)

```bash
scalingo --app my-r-app run --type console
```


## Understanding this buildpack

### Detection

The platform considers the app as a R app if:
- a file called `run.R`, `app.R`, `plumber.R`, `init.R` or `R-packages` exists
  at the root of your project.

> **Warning**\
> **The platform only installs R. If you need a specific framework such as
`plumber` or `shiny`, you still have to
[ask the buildpack to install it](#installing-r-packages).**

### Deployment workflow

During the *`BUILD`* phase, this buildpack:

1. Downloads R if necessary (when the requested version is not in cache).
2. Compiles and installs R with the following options:
   - `--enable-java=no`
   - `--enable-recommended-packages=no`
   - `--enable-R-shlib=yes`
   - `--without-x`
3. Downloads and installs the requested R packages, if any.
4. Validates the build.

:tada: This process results into a scalable image that includes the
configuration, ready to be packaged into a container.

### Release

By default, the release script creates a `console` process type which is
(currently) ignored by Scalingo.

If you need this `console` process type, add a [`Procfile`](https://doc.scalingo.com/platform/app/procfile)
at the root of your project and add the `console` process type:

```yaml
console: R --no-save
```

Moreover, if a file called `run.R`, `app.R` or `plumber.R` exists at the root
of the project, the `release` script also creates a `web` process type with the
following command:

```bash
R --file=${HOME}/{run,app,plumber}.R --gui-none --no-save
```

Consequently:

- if you don't need the `web` process type, make sure to scale it to zero:
  
  ```bash
  scalingo --app -my-r-app scale web:0
  ```

- if you need to run another command, add (or update) your [`Procfile`](https://doc.scalingo.com/platform/app/procfile)
  to specify how to start your `web` process.


## Customizing the Build

### Installing R packages

The buildpack uses a specific file called `R-packages` to install R packages.

The `R-packages` expects one package name per line, with the corresponding
version separated by a single space character.

Version can be set to `latest` or `*`. It can also be omitted, in which case it
will be interpreted as `latest`.


> **Warning**\
> Due to R internals, there are 2 ways of installing a package:
> 1. If you chose to install the `latest` version of a package, R is able to
     automatically download and install it, along with its dependencies.
> 2. If you need a specific version of a package, R is not able to
     automatically download and install its dependencies. You will have to ask
     the buildpack to install each dependency in the appropriate order.

For example, the following `R-packages` file...
```csv
rlang 1.0.2
ellipsis *
fansi latest
shiny
```

...will result in the following actions:
1. The buildpack will download and install `rlang 1.0.2`,
2. it will automatically install the latest version of `ellipsis` (this package
   depends on `rlang`, previously installed),
3. it will automatically install the latest version of `fansi` (this package
   depends on `grDevices` and `utils`, installed with R),
4. it will automatically install all `shiny` dependencies and then the latest
   version of `shiny`.

Note: when a specific version of a package is required (i.e. not `latest`), the
buildpack tries to keep the compiled package in cache for future deployments.

### Environment

The following environment variables are available for you to tweak your
deployment:

#### `R_VERSION`

Version of R to use.\
Defaults to `4.0.0`
