# hiwalay

> Gamitin ang mga modyul ng pangunahing proseso mula sa proseso ng tagabigay.

Mga proseso: [Renderer](../glossary.md#renderer-process)

Ang modyul ng `remote` ay nagbibigay ng isang simpleng paraan na gumawa ng maki-prosesong komyunikasyon (IPC) sa pagitan ng proaesong tagabigay (pahina ng web) at ng pangunahing proseso.

Sa Electron, ang mga modyul na may kaugnayan sa GUI (tulad ng `dialog`, `menu` atbp.) ay makukuha lamang sa pangunahing proseso, hindi sa prosesong tagabigay. Sa ayos ng paggamit sa kanila mula sa prosesong tagabigay, ang modyul ng `ipc` ay kinakailangan na magpadala ng mga maki-prosesong mensahe sa pangunahing proseso. Kasama ang modyul ng `remote`, maaari mong hingiin ang mga pamamaraan ng mga bagay sa pangunahing proseso nang walang nakakapansin na nakapagpadala ng maki-prosesong mensahe, katulad ng [RMI](http://en.wikipedia.org/wiki/Java_remote_method_invocation) ng Java. Ang isang halimbawa ng paglikha ng isang browser window mula sa isang prosesong tagabigay:

```javascript
const {BrowserWindow} = kailangan('electron').remote
let win = new BrowserWindow({width: 800, height: 600})
win.loadURL('https://github.com')
```

**Note:** Sa kabaligtaran (i-akses ang prosesong tagabigay mula sa pangunahing proseso) maaari mong gamitin ang [webContents.executeJavascript](web-contents.md#contentsexecutejavascriptcode-usergesture-callback).

## Mga bagay ng Remote

Bawat bagay (kasama ang mga punsyon) ay nagbabalik dahil ang modyul ng `remote` ay kumakatawan sa isang bagay sa pangunahing proseso (tinatawag natin itong malayong bagay o malayong punsyon). When you invoke methods of a remote object, call a remote function, or create a new object with the remote constructor (function), you are actually sending synchronous inter-process messages.

In the example above, both `BrowserWindow` and `win` were remote objects and `new BrowserWindow` didn't create a `BrowserWindow` object in the renderer process. Instead, it created a `BrowserWindow` object in the main process and returned the corresponding remote object in the renderer process, namely the `win` object.

**Note:** Only [enumerable properties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Enumerability_and_ownership_of_properties) which are present when the remote object is first referenced are accessible via remote.

**Note:** Arrays and Buffers are copied over IPC when accessed via the `remote` module. Modifying them in the renderer process does not modify them in the main process and vice versa.

## Lifetime of Remote Objects

Electron makes sure that as long as the remote object in the renderer process lives (in other words, has not been garbage collected), the corresponding object in the main process will not be released. When the remote object has been garbage collected, the corresponding object in the main process will be dereferenced.

If the remote object is leaked in the renderer process (e.g. stored in a map but never freed), the corresponding object in the main process will also be leaked, so you should be very careful not to leak remote objects.

Primary value types like strings and numbers, however, are sent by copy.

## Passing callbacks to the main process

Code in the main process can accept callbacks from the renderer - for instance the `remote` module - but you should be extremely careful when using this feature.

First, in order to avoid deadlocks, the callbacks passed to the main process are called asynchronously. You should not expect the main process to get the return value of the passed callbacks.

For instance you can't use a function from the renderer process in an `Array.map` called in the main process:

```javascript
// main process mapNumbers.js
exports.withRendererCallback = (mapper) => {
  return [1, 2, 3].map(mapper)
}

exports.withLocalCallback = () => {
  return [1, 2, 3].map(x => x + 1)
}
```

```javascript
// renderer process
const mapNumbers = require('electron').remote.require('./mapNumbers')
const withRendererCb = mapNumbers.withRendererCallback(x => x + 1)
const withLocalCb = mapNumbers.withLocalCallback()

console.log(withRendererCb, withLocalCb)
// [undefined, undefined, undefined], [2, 3, 4]
```

As you can see, the renderer callback's synchronous return value was not as expected, and didn't match the return value of an identical callback that lives in the main process.

Second, the callbacks passed to the main process will persist until the main process garbage-collects them.

For example, the following code seems innocent at first glance. It installs a callback for the `close` event on a remote object:

```javascript
require('electron').remote.getCurrentWindow().on('close', () => {
  // window was closed...
})
```

But remember the callback is referenced by the main process until you explicitly uninstall it. If you do not, each time you reload your window the callback will be installed again, leaking one callback for each restart.

To make things worse, since the context of previously installed callbacks has been released, exceptions will be raised in the main process when the `close` event is emitted.

Upang maiwasan ang problema, siguraduhin burahin anumang kaugnayan sa mga binalikang tawag na ipinasa sa pangunahing proseso. This involves cleaning up event handlers, or ensuring the main process is explicitly told to deference callbacks that came from a renderer process that is exiting.

## Accessing built-in modules in the main process

The built-in modules in the main process are added as getters in the `remote` module, so you can use them directly like the `electron` module.

```javascript
const app = require('electron').remote.app
console.log(app)
```

## Pamamaraan

The `remote` module has the following methods:

### `remote.require(modyul)`

* `modyul` Lubid

Returns `any` - The object returned by `require(module)` in the main process. Modules specified by their relative path will resolve relative to the entrypoint of the main process.

halimbawa.

    proyekto/
    ├── pangunahing
    │   ├── foo.js
    │   └── index.js
    ├── package.json
    └── tagabigay
        └── index.js
    

```js
// main process: main/index.js
const {app} = require('electron')
app.on('ready', () => { /* ... */ })
```

```js
// some relative module: main/foo.js
module.exports = 'bar'
```

```js
// renderer process: renderer/index.js
const foo = require('electron').remote.require('./foo') // bar
```

### `remote.getCurrentWindow()`

Returns [`BrowserWindow`](browser-window.md) - The window to which this web page belongs.

### `remote.getCurrentWebContents()`

Returns [`WebContents`](web-contents.md) - The web contents of this web page.

### `remote.getGlobal(name)`

* `name` String

Returns `any` - The global variable of `name` (e.g. `global[name]`) in the main process.

## Mga Katangian

### `remote.process`

The `process` object in the main process. This is the same as `remote.getGlobal('process')` but is cached.