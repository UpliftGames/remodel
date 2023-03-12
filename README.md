This is a fork of Remodel with changes specific to the workflow at Uplift Games.

Changes from upstream Remodel:
* Uses Luau instead of Lua
* Adds runCommand to run spawn executables and read results
* Adds task library
* Adds httpRequest method
* Makes most remodel functions async
* Fixes ref cloning to match Roblox behavior
* Adds the ability to read Ref properties from Luau
* Adds Font support
* Adds Model Scale support
* Adds Font and Gui Inset migrations

<details><summary>Release Instructions</summary>

New Uplift Games-specific releases should:
* Be created on the `uplift-games-fork-releases` branch (this is like our `main`)
* Be tagged with an appropriate semver **plus** a pre-release tag in the following format:\
  `v1.2.3-uplift.release.1`\
  ...where `v1.2.3` is the semver and `uplift.release.1` increments with each release under that semver.\
  **This tag should be created locally and pushed to kick off automated builds
  (see *Notes on version tags*)**
* The chosen semver should be *relative to upstream according to the difference
  at that release.*\
  For example, if upstream is on `v1.0.0` and we make a minor
  change, we'll be on `v1.0.1-uplift.release.1`. If we make another minor
  change, we'll be on `v1.0.1-uplift.release.2` *because we are still only minor
  changes away from upstream*. This way, if our changes get upstreamed, we won't
  be going backwards in semver.
* Add our changes to `CHANGELOG.md`. If we rebase on a
  new version of Remodel that includes some of our additions, we should list only
  what has changed between upstream Remodel and our fork.
* Where possible, our changes should become PRs to the upstream Remodel repo. When
  we do this, we should include a link to the PR in the changelog entry.

Notes on version tags:
* Tags can be created locally with the command `git tag v1.2.3-uplift.release.1`
* Tags can be pushed to the remote with the command `git push origin v1.2.3-uplift.release.1`
* When a tag starting with `v` is pushed to this repo, an action is kicked off
  which creates a release draft and attached build artifacts when they're
  completed. Go to the releases page and edit the draft to publish it.

</details>

---

<img src="builderman.png" alt="Remodel Logo" width="327" height="277" align="right" />
<h1 align="left">Remodel</h1>

Remodel is a command line tool for manipulating Roblox files and the instances contained within them. It's a scriptable tool designed to enable workflows where no other tool will do.

Remodel can be used to do almost anything with Roblox files. Some uses include:
* [Extracting models from a place to use with Rojo](examples/02-extract-models.lua)
* [Copying terrain from one place into another](examples/04-move-terrain.lua)
* Minifying scripts before deploying a place
* Automatically attaching build metadata to a place
* Synchronizing development places with production

Remodel is still in early development, but much of its API is already stable. Feedback is welcome!

## Installation

The following instructions are for installing the Uplift fork of remodel. If
you're looking to install standard Remodel, see [Remodel's Installation
section](https://github.com/rojo-rbx/remodel#installation).

### With [Aftman](https://github.com/LPGhatguy/aftman)
Remodel can be installed with Aftman, a toolchain manager for Roblox projects:

```toml
[tools]
remodel = "UpliftGames/remodel@0.12.0-uplift.release.12"
```

### From GitHub Releases
You can download pre-built binaries from [the GitHub Releases page](https://github.com/UpliftGames/remodel/releases).

## Quick Start
Most of Remodel's interface is its Lua API. Users write Luau scripts that Remodel runs, providing them with a special set of APIs.

One use for Remodel is to break a place file apart into multiple model files. Imagine we have a place file named `my-place.rbxlx` that has some models stored in `ReplicatedStorage.Models`.

We want to take those models and save them to individual files in a folder named `models`.

```lua
local game = remodel.readPlaceFile("my-place.rbxlx")

-- If the directory does not exist yet, we'll create it.
remodel.createDirAll("models")

local Models = game.ReplicatedStorage.Models

for _, model in ipairs(Models:GetChildren()) do
	-- Save out each child as an rbxmx model
	remodel.writeModelFile("models/" .. model.Name .. ".rbxmx", model)
end
```

For more examples, see the [`examples`](examples) folder.

## Supported Roblox API
Remodel supports some parts of Roblox's API in order to make code familiar to existing Roblox users.

* `Instance.new(className)` (0.5.0+)
	* The second argument (parent) is not supported by Remodel.
* `<Instance>.Name` (read + write)
* `<Instance>.ClassName` (read only)
* `<Instance>.Parent` (read + write)
* `<Instance>:Destroy()` (0.5.0+)
* `<Instance>:Clone()` (0.6.0+)
* `<Instance>:GetChildren()`
* `<Instance>:GetDescendants()` (0.8.0+)
* `<Instance>:FindFirstChild(name)`
	* The second argument (recursive) is not supported by Remodel.
* `<DataModel>:GetService(name)` (0.6.0+)

## Task API
Remodel supports some parts of Roblox's Task API.

* `task.spawn(function | thread, ...): thread`
* `task.defer(function | thread, ...): thread`
* `task.delay(duration: number, function | thread, ...): thread`
* `task.wait(duration: number): number`

Remodel will run until all tasks that it's aware of are complete.\
Remodel's coroutine handling looks a lot like Roblox's: async operations yield
the current coroutine until they complete. You can resume or close the coroutine
early. With some minor modifications, libraries like [Promise](https://eryn.io/roblox-lua-promise/) work in remodel.

### Unstable Task API

If remodel is running longer than expected, you likely have a coroutine running
in the background. You can list all running tasks to the output with
`_runtime.debug.list_lua_tasks()`

## Remodel API
Remodel has its own API that goes beyond what can be done inside Roblox.

### `remodel.readPlaceFile`
```
remodel.readPlaceFile(path: string): Instance
```

Load an `rbxlx` file from the filesystem.

Returns a `DataModel` instance, equivalent to `game` from within Roblox.

Throws on error.

### `remodel.readModelFile`
```
remodel.readModelFile(path: string): List<Instance>
```

Load an `rbxmx` or `rbxm` (0.4.0+) file from the filesystem.

Note that this function returns a **list of instances** instead of a single instance! This is because models can contain mutliple top-level instances.

Throws on error.

### `remodel.readPlaceAsset` (0.5.0+)
```
remodel.readPlaceAsset(assetId: string): Instance
```

Reads a place asset from Roblox.com, equivalent to `remodel.readPlaceFile`.

**This method requires web authentication for private assets! See [Authentication](#authentication) for more information.**

Throws on error.

### `remodel.readModelAsset` (0.5.0+)
```
remodel.readModelAsset(assetId: string): List<Instance>
```

Reads a model asset from Roblox.com, equivalent to `remodel.readModelFile`.

**This method requires web authentication for private assets! See [Authentication](#authentication) for more information.**

Throws on error.

### `remodel.writePlaceFile`
```
remodel.writePlaceFile(path: string, instance: DataModel)
```

Saves an `rbxlx` file out of the given `DataModel` instance.

If the instance is not a `DataModel`, this method will throw. Models should be saved with `writeModelFile` instead.

Throws on error.

### `remodel.writeModelFile`
```
remodel.writeModelFile(path: string, instance: Instance)
```

Saves an `rbxmx` or `rbxm` (0.4.0+) file out of the given `Instance`.

If the instance is a `DataModel`, this method will throw. Places should be saved with `writePlaceFile` instead.

Throws on error.

### `remodel.writeExistingPlaceAsset` (0.5.0+)
```
remodel.writeExistingPlaceAsset(instance: Instance, assetId: string)
```

Uploads the given `DataModel` instance to Roblox.com over an existing place.

If the instance is not a `DataModel`, this method will throw. Models should be uploaded with `writeExistingModelAsset` instead.

**This method always requires web authentication! See [Authentication](#authentication) for more information.**

Throws on error.

### `remodel.writeExistingModelAsset` (0.5.0+)
```
remodel.writeExistingModelAsset(instance: Instance, assetId: string)
```

Uploads the given instance to Roblox.com over an existing model.

If the instance is a `DataModel`, this method will throw. Places should be uploaded with `writeExistingPlaceAsset` instead.

**This method always requires web authentication! See [Authentication](#authentication) for more information.**

Throws on error.

### `remodel.getRawProperty` (0.6.0+)
```
remodel.getRawProperty(instance: Instance, name: string): any?
```

Gets the property with the given name from the given instance, bypassing all validation.

This is intended to be a simple to implement but very powerful API while Remodel grows more ergonomic functionality.

Throws if the value type stored on the instance cannot be represented by Remodel yet. See [Supported Roblox Types](#supported-roblox-types) for more details.

### `remodel.setRawProperty` (0.6.0+)
```
remodel.setRawProperty(
	instance: Instance,
	name: string,
	type: string,
	value: any,
)
```

Sets a property on the given instance with the name, type, and value given. Valid values for `type` are defined in [Supported Roblox Types](#supported-roblox-types) in the left half of the bulleted list.

This is intended to be a simple to implement but very powerful API while Remodel grows more ergonomic functionality.

Throws if the value type cannot be represented by Remodel yet. See [Supported Roblox Types](#supported-roblox-types) for more details.

### `remodel.readFile` (0.3.0+)
```
remodel.readFile(path: string): string
```

Reads the file at the given path.

Throws on error, like if the file did not exist.

### `remodel.readDir` (0.4.0+)
```
remodel.readDir(path: string): List<string>
```

Returns a list of all of the file names of the children in a directory.

Throws on error, like if the directory did not exist.

### `remodel.writeFile` (0.3.0+)
```
remodel.writeFile(path: string, contents: string)
```

Writes the file at the given path.

Throws on error.

### `remodel.removeFile`
```
remodel.removeFile(path: string)
```

Removes the file at the given path.

This is a thin wrapper around Rust's [`fs::remove_file`](https://doc.rust-lang.org/std/fs/fn.remove_file.html) function.

Throws on error.

### `remodel.createDirAll`
```
remodel.createDirAll(path: string)
```

Makes a directory at the given path, as well as all parent directories that do not yet exist.

This is a thin wrapper around Rust's [`fs::create_dir_all`](https://doc.rust-lang.org/std/fs/fn.create_dir_all.html) function. Similar to `mkdir -p` from Unix.

Throws on error.

### `remodel.removeDir`
```
remodel.removeDir(path: string)
```

Removes a directory at the given path.

This is a thin wrapper around Rust's [`fs::remove_dir_all`](https://doc.rust-lang.org/std/fs/fn.remove_dir_all.html) function.

Throws on error.

### `remodel.isFile` (0.7.0+)
```
remodel.isFile(path: string): bool
```

Tells whether the given path is a file.

This is a thin wrapper around Rust's [`fs::metadata`](https://doc.rust-lang.org/std/fs/fn.metadata.html) function.

Throws on error, like if the path does not exist.

### `remodel.isDir` (0.7.0+)
```
remodel.isDir(path: string): bool
```

Tells whether the given path is a directory.

This is a thin wrapper around Rust's [`fs::metadata`](https://doc.rust-lang.org/std/fs/fn.metadata.html) function.

Throws on error, like if the path does not exist.

### `remodel.runCommand` (0.12.0+)
```
remodel.runCommand(command: string, args: { string }?): { stdout: string, stderr: string, status: boolean, code: number }
```

Runs a command and returns the result.

### `remodel.httpRequest` (0.12.0+)
```
type HttpRequestOptions = {
	Url: string,
	Method: string?,
	Headers: { [string]: string }?,
	Body: string?,

	RobloxAuth: boolean?,
	RobloxApiKey: boolean?,
	RobloxCsrf: boolean?,
}

type HttpRequestResponse = {
	Success: boolean,
	StatusCode: number,
	StatusMessage: string,
	Headers: { [string]: string },
	Body: string,
}

remodel.httpRequest(options: HttpRequestOptions): HttpRequestResponse
```

Sends an http request and returns the result.

Throws if your request is malformed or if there was an issue _sending_ the
request. If the request succeeded, but the server did not like it, the response
is still returned without throwing.

Options notes:
* `Method` should be upper-case, and defaults to `GET`
* `RobloxAuth` will add a `Cookie` header with the current Roblox auth cookie
* `RobloxApiKey` will add a `x-api-key` header with the open cloud API key
* `RobloxCsrf` will automatically retry with the `x-csrf-token` header if given
  a csrf challenge.

## JSON API

### `json.fromString` (0.7.0+)
```
json.fromString(source: string): any
```

Decodes a string containing JSON.

Throws on error, like if the input JSON is invalid.

### `json.toString` (0.7.0+)
```
json.toString(value: any): string
```

Encodes a Lua object as a JSON string. Can only encode Lua primitives like tables, strings, numbers, bools, and nil. Instances cannot be encoded to JSON.

Throws on error, like if the input table cannot be encoded.

### `json.toStringPretty` (Unreleased)
```
json.toStringPretty(value: any, indent?: string = "  "): string
```

Encodes a Lua object as a prettified JSON string. If an indent is passed, will use that for indentation, otherwise will default to two spaces.

Throws on error, like if the input table cannot be encoded.

## Supported Roblox Types
When interacting with Roblox instances, Remodel doesn't support all value types yet and may throw an error.

Supported types and their Lua equivalents:

* `String`: `string`
* `Content`: `string`
* `Bool`: `boolean`
* `Float64`: `number`
* `Float32`: `number`
* `Int64`: `number`
* `Int32`: `number`

More types will be added as time goes on, and Remodel will slowly begin to automatically infer correct types in more contexts.

## Authentication
Some of Remodel's APIs access the Roblox web API and need authentication in the form of a `.ROBLOSECURITY` cookie to access private assets. Auth cookies look like this:

```
_|WARNING:-DO-NOT-SHARE-THIS.--Sharing-this-will-allow-someone-to-log-in-as-you-and-to-steal-your-ROBUX-and-items.|<actual cookie stuff here>
```

**Auth cookies are very sensitive information! If you're using Remodel on a remote server like Travis CI or GitHub Actions, you should create a throwaway account with limited permissions in a group to ensure your valuable accounts are not compromised!**

On Windows, Remodel will attempt to use the cookie from a logged in Roblox Studio session to authenticate all requests.

To give a different auth cookie to Remodel, use the `--auth` argument:

```
remodel run foo.lua --auth "$MY_AUTH_COOKIE"
```

You can also define the `REMODEL_AUTH` environment variable to avoid passing `--auth` as an argument.

## Remodel vs rbxmk
Remodel is similar to [rbxmk](https://github.com/Anaminus/rbxmk):
* Both Remodel and rbxmk use Lua
* Remodel and rbxmk have a similar feature set and use cases
* Remodel is imperative, while rbxmk is declarative
* Remodel emulates Roblox's API, while rbxmk has its own, very unique API

## License
Remodel is available under the terms of the MIT license. See [LICENSE.txt](LICENSE.txt) for details.

[Logo source](https://pixabay.com/illustrations/factory-worker-industry-2318026/).
