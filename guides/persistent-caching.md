Welcome to the persistent caching guide.

# Opt-in

First, note that persistent caching is not enabled by default. You have to opt-in using it.

Why is that?
webpack tries to favor safety over performance.
We don't want to enable a feature by default that will improve your performance by 95% but breaks your application/workflow/build in 5%.

That sounds like it's broken, but believe me, it's not.
But it requires an extra step by the developer to configure it correctly.

While serialization and deserialization would work out-of-the-box without extra steps by the developer, the part that may not work out-of-the-box is cache invalidation.

What's cache invalidation?
webpack needs to figure out when cache entries are no longer valid and stop using them for the build.
So this happens when you change a file in your application.

Example: You change `magic.js`.
webpack must invalidate the cache entry for `magic.js`.
The build will process the file again, i. e. runs babel, typescript, whatever, parses the file and runs Code Generation again.
Then webpack probably also invalidates the cache entry for `bundle.js`,
and the build will rebuild this file from the contained modules.

For this webpack tracks `fileDependencies` `contextDependencies` and `missingDependencies` for each Module, and creates a file system snapshot.
This snapshot is compared with the real file system and when differences are detected a re-build is triggered for that module.

Then for the cache entry of `bundle.js`, webpack stores an `etag`, which is a hash of all contributors.
This `etag` is compared and only when it matches the cache entry can be used.

All this was also required for in-memory caching in webpack 4.
And all this works out-of-the-box from developer-view without extra configuration.
For persistent caching in webpack 5, there is a new challenge.

webpack also needs to invalidate cache entries:
* when you npm upgrade a loader or plugin
* when you change your configuration
* when you change a file that is being read in the configuration
* when you npm upgrade a dependency that is used in the configuration
* when you pass different command-line arguments to your build script
* when you have a custom build script and change that

Here it becomes tricky.
webpack is not able to handle all these cases out-of-the-box.
That's why we've chosen the safe way and made persistent caching an opt-in feature.
We want you to learn how to enable persistent caching to give you the correct hints.
We want you to know which configuration needs to be used to handle i. e. your custom build script.

# Build dependencies, version and name

To handle these "dependencies" of your build webpack provides three new tools:

## Build dependencies

This is a new configuration option `cache.buildDependencies`, which allows specifying code dependencies of the build process.
To make it easier webpack takes care of the resolving and following of dependencies of specified values.

There are two possible types of values: files and directories.
Directories must end with a slash. Everything else is resolved as a file.

For directories, the nearest `package.json` is analysed for dependencies.
For files, we will look into the Node.js module cache to find dependencies.

Example: The build usually depends on the `lib` folder of `webpack` itself.
You could specify it this way:

``` js
cache.buildDependencies: {
    defaultWebpack: ["webpack/lib/"]
}
```

This invalidates the persistent cache when anything in `webpack/lib` or in dependencies of webpack like `watchpack`, `enhanced-resolved`, etc. changes.
Coincidentally this is already the default value, so you don't have to specify it.

Another example: The build usually also depends on your configuration file.
You could specify it this way:

``` js
cache.buildDependencies: {
    config: [__filename]
}
```

The `__filename` variable points to the current file in Node.js.

This invalidates the persistent cache when your config or anything the config depends on via `require()` changes.
As your config probably references all used plugins via `require()` they also become build dependencies.

If your configuration file reads a file via `fs.readFile`, this would **not** become a build dependency, as webpack only follows `require()`.
You need to add such files to `buildDependencies` manually.

## Version

Some dependencies of your build can't be expressed as references to a file, i. e. values read from a database, environment variables or values passed on the command line.
For these values, there is a new configuration option `cache.version`.

`cache.version` is a string. Passing a different string will invalidate the persistent cache.

Example: Your config reads the environment variable `GIT_REV` and uses this value with the `DefinePlugin` to embed it into the bundle.
This makes `GIT_REV` a dependency for your build.
You could specify it this way:

``` js
cache: {
    version: `${process.env.GIT_REV}`
}
```

## Name

In some cases, dependencies toggle between multiple different values and invalidating the persistent cache for each value change would be wasteful.
For these values, there is a new configuration options `cache.name`.

`cache.name` is a string. Passing a value will create a separate independent persistent cache.

`cache.name` is used as the filename of the persistent cache file. Make sure to only pass short and fs-safe names.

Example: Your config uses the `--env.target mobile|desktop` argument to creates builds for either mobile or desktop users.
You could specify it this way:

``` js
cache: {
    name: `${env.target}`
}
```

# Performance optimizations

Hashing and timestamping a large part of `node_modules` for build and normal dependencies would be pretty expensive, and would slow down webpack a lot.
To avoid this webpack includes a performance optimization that skips over files in `node_modules` by default and uses the `version` and `name` in `package.json` as a source of truth.

This optimization will be used for all paths in the configuration option `snapshot.managedPaths`.
It defaults to the `node_modules` directory in which webpack is installed.

**Do not edit `node_modules` by hand** when this optimization is enabled.
You can disable it with `cache.managedPaths: []`.

When using Yarn PnP another optimization kicks in.
All files in the yarn cache are skipped for hashing and timestamping at all (not even `version` and `name` is tracked).
This is valid as Yarn uses hashes in the file path, so the content of the files is actually immutable and will never change.

This is controlled by the configuration option `snapshot.immutablePaths`.
It defaults to the yarn cache in which webpack is installed when Yarn PnP is enabled.

We could tell you to not edit the yarn cache by hand, but this is a no-go anyway.

# Snapshot types

When none of the `snapshot.managedPaths` or `snapshot.immutablePaths` optimization kicks in webpack creates a snapshot, which will later compared with the (updated?) state of the filesystem.
For different use cases there are different types of snapshots:

* Timestamp: The modified timestamp is read from filesystem, stored and compared.
* Contenthash: The content of the file is read from filesystem, a hash is stored and compared.
* Timestamp + Contenthash: The modified timestamp and the content of the file is read from filesystem, the timestamp and a hash of the content is stored. The timestamp is compared first and on missmatch the hash is compared.

Timestamping is very fast to capture and compare as it only needs a meta data read from filesystem. Contenthashing is slow compared to that, because it need to read the file content from filesystem. But timestamping might invalidate the cache even when the file content is the same, but only the timestamp changed. This can happen because of multiple factors:

* git operations always update the timestamp. e. g. fresh clone, changing branches
* save without changes

So the question is: Do you expect timestamp changes, while content stays equal? And is the extra performance cost worth it?

"Timestamp" makes sense when we expect timestamps to stay equal and the invalidation cost is not that high. Here the faster capture and compare is preferable over rare additional invalidations.

"Timestamp + Contenthash" makes sense when we expect timestamps to stay equal, but invalidation cost is so high, that we don't want to risk additional invalidations. The slower capturing is a one time cost we need to accept here. Comparing is usually fast as we expect timestamps to stay equal.

"Contenthash" makes sense when we expect timestamps to not stay equal. This saves capturing and comparing the timestamp, that won't be equal anyway.

We also need to look at different operations in webpack that need snapshots and how much a cache invalidation costs.

Resolving (`snapshot.resolve`): Snapshots are used to determine if a request need to be resolved again.
When the cache can't be used the resolving algorithm runs and figures out the resulting file the request points to.

Modules (`snapshot.module`): Snapshots are used to determine if a module need to be rebuild.
When the cache can't be used the loaders has to be executed again and the result has to be parsed.

Resolving build dependencies (`snapshot.resolveBuildDependencies`): Snapshots are used to determine if a build dependency request need to be resolved again.
When the cache can't be used the resolving algorithm runs to figure out where the build dependency is on disk.

Build Dependencies (`snapshot.buildDependencies`): Snapshots are used to determine if build dependencies have changed.
When the cache can't be used the whole persistent cache will be invalidated as the inner workings of the build have changed.

We see that different operations have different costs on cache invalidation. We need to consider that when choosing the snapshot type.

Scenario Local development computer:

Here we would expect timestamps to usually be equal. So using "Timestamp" for resolving and modules makes sense.
For (resolving) build dependencies "Timestamp + Contenthash" makes sense, as an unnecessary invalidation would be very costy.

Scenario CI build:

As CI build usually create a fresh git clone of the source code, we expect timestamps to not be equal.
So "Contenthash" makes sense for all operations here

Scenario CI build with update:

Some CI Servers can be configured to only run a "git pull" instead of a full clone. In this case timestamps would stay equal and the "Timestamp" resp. "Timestamp + Contenthash" method would make sense (similar to a local development computer).

# Use the persistent cache

Make sure you have read and understood the information above!

Here is a typical config to enable the persistent cache:

``` js
cache: {
    type: "filesystem",
    buildDependencies: {
        config: [ __filename ] // you may omit this when your CLI automatically adds it
    }
}
```

## Watching

The persistent cache can be used for single builds and continuous building (watch).

When setting `cache.type: "filesystem"` webpack internally enables the filesystem cache and the memory cache in a layered way.
Reading from cache will look into the memory cache first and fallback to filesystem cache.
Writing to cache will write to both caches.

The filesystem cache won't directly serialize write requests to disk. It will wait until the compilation process has finished and the compiler is idling.
This happens because serialization and disk writing uses up resources and we don't want to add delay the compilation process.

For single builds the workflow is:

* Loading cache
* Building
* Emitting
* Display results (stats)
* Persisting cache (if changed)
* Process exits

For continuous builds (watch) the workflow is:

* Loading cache
* Building
* Emitting
* Display results (stats)
* Attach filesystem watchers
* Wait `cache.idleTimeoutForInitialStore`
* Persisting cache (if changed)
* On change:
  * Building
  * Emitting
  * Display results (stats)
  * Wait `cache.idleTimeout`
  * Persisting cache (if changed)

You see the two new configuration options `cache.idleTimeout` and `cache.idleTimeoutForInitialStore` which control how long the compiler has to idle before cache is persisted.
`cache.idleTimeout` defaults to 60s and `cache.idleTimeoutForInitialStore` defaults to 0s.
As serialization blocks the event loop, no change is detected while the cache is serialized.
The delay tries to avoid recompilation delay in watch mode due to fast-paced editing of files while trying to keep the persistent cache fresh for the next cold start.
It's a trade-off, feel free to choose a value that fits your workflow. Smaller values shorten cold startup time and increase the risk of delayed rebuilds. Bigger values can increase cold startup time and reduce the risk of delayed rebuilds.

## Error handling

The persistent cache recovers from any error by either dropping the whole cache and doing a fresh build or by omitting the offending cache entry and leaving this item uncached.

Warnings will be emitted in such cases with the webpack infrastructure logger.
See `infrastructureLogging` configuration option for details.
By default they will be printed to stderr.

## Garbage collection

Unused cache entries are removed from the cache after 1 month.

## Splitting in files

The persistent caching system will automatically distribute cache entries into cache files aiming for the following:

* Unused and used cache entries should not be together in a cache file.
* Cache files should be at least 1 MB big.
* A maximum of 50,000 fresh cache items are together in a cache file.

After a short while of usage this should result in:

* No cache items are deserialized that are not used
* Cache items are grouped to benefit from object deduplication
* Too big files are not created that would be expensive to rewrite

---

# Details

The following information is not needed for normal usage.

## Guideline for higher-level tools using webpack

A tool that wraps webpack may choose different defaults.
When it doesn't allow to use a custom config to extend webpack, it could switch the persistent cache on by default as it has full control over all build dependencies.

## Guideline for CLIs

A CLI using webpack could add some build dependencies by default, which is not possible for the webpack itself.

* It should set `cache.buildDependencies.defaultConfig` to the used config file by default.
* It should append the command line arguments to `cache.version`
* It may add a note to `cache.name` when command line arguments are used

## Debug information

With the following config additional debug information will be emitted:

``` js
infrastructureLogging: {
    debug: /webpack\.cache/
}
```

## Internal workflow

* webpack reads the cache file.
  * There is no cache file -> build without cache
  * `version` in the cache file doesn't match the `cache.version` -> build without cache
* webpack compares the `resolve snapshot` with the filesystem
  * It does match -> continue below
  * It doesn't match:
    * Resolve all `resolve results` again
      * It doesn't match -> build without cache
      * It does match -> continue below
* webpack compares the `build dependencies snapshot` with the filesystem
  * It doesn't match -> build without cache
  * It does match -> continue below
* cache metadata is deserialized
* build runs (with or without cache)
  * accessing cache entries causes cache files to be deserialized
  * build dependencies are tracked
    * from `cache.buildDependencies`
    * from used loaders
    * defaults
* new build dependencies are resolved
  * `resolve dependencies` are tracked
  * `resolve results` are tracked
* a snapshot from all new `resolve dependencies` is created
  * If there is already a snapshot for that, both are merged
* a snapshot from all new `build dependencies` is created
  * If there is already a snapshot for that, both are merged
* persistent cache files are serialized to disk

## Serialization

All classes that should support serialization need to have a serializer registered:

``` js
webpack.util.serialization.register(Constructor, request, name, serializer);
```

`Constructor` should be a class or constructor function.
For any object that should be serialized `object.constructor` will be used to find a serializer.

`request` will be used to load the module that calls `register`.
It should point to the current module.
It will be used this way: `require(request)`.

`name` is used to differentiate multiple `register` calls with the same `request`.

`serializer` is an object with at least two methods `serialize` and `deserialize`.

`serializer.serialize(object, context)` is called when an object should be serialized.
`context` is an object that contains at least a `write(anything)` method.
This method writes something into the output stream.
The passed value will be serialized too.

`serializer.deserialize(context)` is called when a serialized object should be deserialized.
`context` is an object that contains at least a `read(): anything` method.
This method deserializes something from the input stream.
`deserialize` must return the deserialized object.

`serialize` and `deserialize` should read and write the same objects in the same order.

Example:

``` js
// some-module/lib/MyClass.js
class MyClass {
    constructor(a, b) {
        this.a = a;
        this.b = b;
        this.c = undefined;
    }
}

register(MyClass, "some-module/lib/MyClass", null, {
    serialize(obj, { write }) {
        write(obj.a);
        write(obj.b);
        write(obj.c);
    },
    deserialize({ read }) {
        const obj = new MyClass(read(), read());
        obj.c = read();
        return obj;
    }
});
```

You can expect serializer for primitive types and basic javascript classes to be already registered, i. e. string, number, Array, Set, Map, RegExp, plain objects, Error.
