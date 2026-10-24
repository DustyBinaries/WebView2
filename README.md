# WebView2 for Delphi 7

[Português](README_pt-BR.md)

**Microsoft Edge WebView2** in a **Delphi 7** VCL control: `TWebView2Browser`.

Delphi 7 has no `TEdgeBrowser` (it only exists from 10.4 on). So the WebView2 COM interfaces were
declared by hand in `WebView2.pas`, from the `WebView2.h` of the official `Microsoft.Web.WebView2`
NuGet package, and wrapped in the control.

## Requirements

- Delphi 7 (produces 32-bit executables).
- **WebView2 Runtime** installed. It ships with Windows 11; on Windows 10 use the "Evergreen" installer:
  <https://developer.microsoft.com/microsoft-edge/webview2/>
- **`WebView2Loader.dll`, x86 version**, in the executable's folder. It is in this folder; the
  examples' `build.bat` files copy it next to the exe (see [Origin of `WebView2Loader.dll`](#origin-of-webview2loaderdll)).

No external libraries are required.

### Origin of `WebView2Loader.dll`

The DLL in this folder is the one from `build\native\x86\` in Microsoft's official NuGet package, unmodified:

| | |
|---|---|
| Package | [`Microsoft.Web.WebView2`](https://www.nuget.org/packages/Microsoft.Web.WebView2) **1.0.4258.31** |
| Path in the package | `build/native/x86/WebView2Loader.dll` |
| File version | 1.0.4258.31 |
| Signature | Authenticode, `CN=Microsoft Corporation` |
| SHA-256 | `87C61E2CAE380112C60853E28178B8EA9A0D63739FC03CEAD680D4AD3FDE6B7D` |

To verify it or download it again (the `.nupkg` is a zip):

```powershell
Invoke-WebRequest https://www.nuget.org/api/v2/package/Microsoft.Web.WebView2/1.0.4258.31 -OutFile webview2.zip
Expand-Archive webview2.zip webview2
Get-FileHash webview2\build\native\x86\WebView2Loader.dll        # must match the SHA-256 above
Get-AuthenticodeSignature webview2\build\native\x86\WebView2Loader.dll
```

To update, change the version in the URL, copy the new DLL into this folder and update this table.

## Files

| File | What it is |
|---|---|
| `WebView2.pas` | WebView2 COM interfaces and dynamic loading of `WebView2Loader.dll` |
| `WebView2Browser.pas` | `TWebView2Browser`, the VCL control that hosts the browser |
| `WebView2Loader.dll` | x86 loader from the official NuGet package |
| `docs/` | WebResourceRequested ([English](docs/WebResourceRequested.md), [português](docs/WebResourceRequested_pt-BR.md)): request interception in detail |
| `Tests/` | DUnit tests (`run-tests.bat`) |
| `Examples/Demo/` | Example with all the control's features (see below) |

To use it in a project, add `WebView2.pas` and `WebView2Browser.pas` (or put this folder in the
*Search path*) and place `WebView2Loader.dll` next to the executable.

## Example

`Examples/Demo/` is a browser with an address bar and a panel to try each feature:
navigation events, URL blocking, new-window policy, `PostWebMessage` /
`ExecuteScript`, a page embedded in the exe and a "local API" with `WebResourceRequested`.

Open `WebView2Demo.dpr` in the IDE, or from the command line:

```bat
Examples\Demo\build.bat
```

`build.bat` generates `WebView2Demo.res` from the `.rc` (if it does not exist yet), **always
regenerates `web.res`** from `web.rc` (the files in the `web\` folder), calls `dcc32` and copies
`WebView2Loader.dll` into the folder. If you compile from the IDE after changing `web\`, run
`brcc32 web.rc` first (or `build.bat`). If Delphi is not in `C:\bin\Borland\Delphi7`, set the
`DELPHI` variable beforehand.

The sources (`.pas`, `.dfm`, `.dpr`) are in **ANSI (cp1252) with CRLF**, as Delphi 7 expects.
Do not save them as UTF-8, or accented characters in strings will break.

## Using `TWebView2Browser`

The control is created at run time (it is not installed on the palette):

```pascal
FBrowser := TWebView2Browser.Create(Self);
FBrowser.Parent := Self;
FBrowser.Align := alClient;
FBrowser.OnInitialized := BrowserInitialized;
FBrowser.CreateWebView;                    // asynchronous: ends in OnInitialized
FBrowser.Navigate('https://example.com');  // can be called earlier; it stays pending
```

### Methods

| Method | Description |
|---|---|
| `CreateWebView` | Creates the environment and the browser (asynchronous). Raises `EWebView2Error` if the loader or the runtime is missing |
| `Ready` | `True` once the browser has been created |
| `Recreate` | Closes the WebView and creates another, returning to the last URL (this is what `OnProcessFailed` does when the browser crashes) |
| `Navigate(Url)` / `NavigateToString(Html)` | Navigates to a URL / loads HTML directly |
| `GoBack`, `GoForward`, `Reload`, `Stop` | Navigation |
| `CanGoBack`, `CanGoForward`, `Source` | Navigation state |
| `ExecuteScript(Js)` | Runs JavaScript; the result (JSON) arrives in `OnScriptResult` |
| `AddScriptOnDocumentCreated(Js)` | Script that runs in every new document, before the page's scripts. Can be called before `CreateWebView` |
| `PostWebMessage(Text)` / `PostWebMessageAsJson(Json)` | Sends a message to the page |
| `OpenDevTools` | Opens DevTools |
| `NotifyParentWindowPositionChanged` | Call it in the form's `WM_MOVE`, so popups and dropdowns open in the right place |
| `ServeEmbeddedResources(Host)` | Serves the exe's `RCDATA` resources at `https://<Host>/` (see below) |
| `AddWebResourceRequestedFilter(Pattern, [Context])` | Makes URLs matching the pattern (`*` wildcard) go through `OnWebResourceRequested` |
| `RemoveWebResourceRequestedFilter(Pattern, [Context])` / `ResourceFilterCount` | Removes a filter (same pattern and context) / how many there are |
| `EmbeddedUrl(Path)` | Builds the URL of an embedded resource, e.g. `https://app.localhost/index.html` |

### Properties

| Property | Description |
|---|---|
| `UserDataFolder` | Cache/cookies folder. Default: `%LOCALAPPDATA%\<exe name>\WebView2` |
| `Language` | WebView2 UI language and `Accept-Language`, e.g. `'pt-BR'`. Before `CreateWebView` |
| `AdditionalBrowserArguments` | Chromium arguments, e.g. `'--disable-gpu'`. Before `CreateWebView` |
| `WebView` / `Controller` | The COM interfaces, for whatever the control does not wrap |

### Events

| Event | When |
|---|---|
| `OnInitialized(ErrorCode)` | The browser was created (or failed: `ErrorCode` is the HRESULT) |
| `OnNavigationStarting(Uri, IsUserInitiated, IsRedirected, var Cancel)` | Before each navigation. `Cancel := True` prevents loading |
| `OnNavigationCompleted(Success, WebErrorStatus)` | The navigation finished |
| `OnNewWindowRequested(Uri, IsUserInitiated, var Action)` | The page asked for a new window (`target="_blank"`, `window.open`). `Action` decides what happens |
| `OnSourceChanged(Url)` | The current URL changed |
| `OnTitleChanged(Title)` | `document.title` changed |
| `OnWebMessageReceived(Msg)` | The page called `chrome.webview.postMessage(...)` |
| `OnWebMessageJson(Json, Source, var Handled)` | Before the previous one: the message as JSON and the source URL. `Handled := True` suppresses it. Useful for implementing a protocol |
| `OnAcceleratorKeyPressed(KeyEventKind, VirtualKey, var Handled)` | A key pressed while the page has focus, before it reaches the JS (the keyboard goes straight to the WebView2 process, and the VCL does not see it). `KeyEventKind`: `COREWEBVIEW2_KEY_EVENT_KIND_KEY_DOWN`, `_KEY_UP`, `_SYSTEM_KEY_DOWN`, `_SYSTEM_KEY_UP`. `Handled := True` stops the page from receiving the key. Useful for menu shortcuts |
| `OnScriptResult(ErrorCode, ResultJson)` | Result of an `ExecuteScript` |
| `OnWebResourceRequested(Request, Response, var Handled)` | A request matched a filter. `Handled := True` answers with `Response`; otherwise it goes on to the network |
| `OnProcessFailed(Kind, var Action)` | A WebView2 process crashed or hung. `Action` comes with the default already set (see [Robustness](#robustness)) |
| `OnCallbackError(E)` | An exception in one of the events above. Without a handler, it goes to the application's default handler |

#### Filtering navigations with `OnNavigationStarting`

```pascal
procedure TfrmMain.BrowserNavigationStarting(Sender: TObject; const Uri: WideString;
  IsUserInitiated, IsRedirected: Boolean; var Cancel: Boolean);
begin
  Cancel := Pos('blocked', LowerCase(Uri)) > 0;
end;
```

A cancelled navigation **still fires `OnNavigationCompleted`**, with `Success = False` and
`WebErrorStatus = COREWEBVIEW2_WEB_ERROR_STATUS_OPERATION_CANCELED` (14). Do not treat this code
as an error.

#### Controlling new windows with `OnNewWindowRequested`

Unhandled, WebView2 opens `target="_blank"` links and `window.open()` in its own popup window,
outside the application's control. The event lets you choose another behavior through the
`Action` parameter:

| `Action` | Effect |
|---|---|
| `nwaPopup` (default) | WebView2 opens its own popup window |
| `nwaSameWindow` | Navigates to the URL in the `TWebView2Browser` itself |
| `nwaCancel` | Opens nothing. Use it when the handler takes care of the URL itself |

```pascal
procedure TfrmMain.BrowserNewWindowRequested(Sender: TObject; const Uri: WideString;
  IsUserInitiated: Boolean; var Action: TWV2NewWindowAction);
begin
  // open in the Windows default browser
  Action := nwaCancel;
  ShellExecute(Handle, 'open', PChar(string(Uri)), nil, nil, SW_SHOWNORMAL);
end;
```

With `nwaSameWindow`, the navigation goes through `OnNavigationStarting` as usual, so filters
still apply.

## Page embedded in the executable

The interface's HTML, CSS, JS and images live **inside the `.exe`**, as `RCDATA` resources. You
only need to distribute the executable and `WebView2Loader.dll`.

The control creates a virtual host and intercepts requests to it (`WebResourceRequested`),
answering with the contents of the resources:

```pascal
FBrowser.ServeEmbeddedResources('app.localhost');          // before or after CreateWebView
FBrowser.CreateWebView;
FBrowser.Navigate(FBrowser.EmbeddedUrl('index.html')); // https://app.localhost/index.html
```

Since everything is served by URL, the page uses relative paths as usual
(`<link href="style.css">`, `<script src="app.js">`, `<img src="img/logo.svg">`) and runs in an
`https://` origin, so APIs that require a secure context work.

**Resource name:** the URL path in uppercase, with everything that is not a letter or digit
replaced by `_`. The root (`/`) serves `INDEX_HTML`.

| URL | Resource in `web.rc` |
|---|---|
| `https://app.localhost/index.html` | `INDEX_HTML RCDATA "web\\index.html"` |
| `https://app.localhost/style.css` | `STYLE_CSS RCDATA "web\\style.css"` |
| `https://app.localhost/img/logo.svg` | `IMG_LOGO_SVG RCDATA "web\\img\\logo.svg"` |

To add a file: put it in `web\`, add the line to `web.rc` and recompile.
The `Content-Type` is chosen by extension (html, css, js, json, svg, png, jpg, gif, ico, woff2). A
path with no matching resource gets **404**, which fires `OnNavigationCompleted` with
`Success = False` if it is the main page.

Limitations: the resource name only has ASCII letters and digits, so accented characters and
spaces become `_` and different names may collide. Prefer simple, ASCII names.

## Intercepting requests: `OnWebResourceRequested`

The event receives requests that match some filter: those registered with
`AddWebResourceRequestedFilter` and the `ServeEmbeddedResources` host, which is filtered
automatically. It runs **before** the embedded-resources server, so you can build a
"local API" on the same host, with no HTTP server:

```pascal
procedure TfrmMain.BrowserWebResourceRequested(Sender: TObject;
  Request: TWV2WebResourceRequest; Response: TWV2WebResourceResponse; var Handled: Boolean);
begin
  if Request.Path = 'api/customers' then
  begin
    Response.SetJson('[{"id":1,"name":"São João Bakery"}]');
    Handled := True;
  end;
  // Handled = False: goes on to the embedded resources (or the network)
end;
```

```js
fetch('api/customers').then(r => r.json()).then(console.log);
```

**`TWV2WebResourceRequest`** (valid only during the event):

| Member | Description |
|---|---|
| `Uri`, `Method` | Full URL and HTTP method |
| `Path` / `RawPath` | `https://app.localhost/a%20b/x?q=1` → `a b/x` (decoded, UTF-8) / `a%20b/x` (as received) |
| `QueryParam(Name)`, `HasQueryParam(Name)` | Decoded value of a parameter (`+` becomes a space) |
| `Query` | The whole query, **not** decoded (decoding it all at once would break `%26` and `%3D`) |
| `GetHeader(Name)` | Reads a request header |
| `SetHeader(Name, Value)`, `RemoveHeader(Name)` | Change the request **that goes on to the network**, e.g. inject `Authorization` |
| `Content` / `ContentText` | Body (POST/PUT): raw bytes / text decoded from UTF-8 |
| `ResourceContext` | Resource type (`COREWEBVIEW2_WEB_RESOURCE_CONTEXT_*`) |

**`TWV2WebResourceResponse`** (starts as `200 OK`, empty):

| Member | Description |
|---|---|
| `StatusCode`, `ReasonPhrase` | HTTP status |
| `SetText(Text, [ContentType])`, `SetJson(Json)` | UTF-8 content, with `charset=utf-8` |
| `SetData`, `LoadFromStream`, `LoadFromFile` | Binary content. `LoadFromFile` infers the `Content-Type` from the extension |
| `SetHeader(Name, Value)` / `Headers` | Response headers |
| `Defer` / `Complete` | Answer later (see below) |

**Deferred response:** to answer later (e.g. when a query finishes), call
`Response.Defer`, set `Handled := True`, keep the response and call `Response.Complete` later,
**on the UI thread**. After `Complete` the object is released. The `Request` does not outlive
the event, so copy what you need from it first.

```pascal
if Request.Path = 'report' then
begin
  Response.Defer;
  Handled := True;
  FPending.Add(Response);   // a timer, or the end of a thread (via Synchronize), calls:
end;                        //   Response.SetJson(...); Response.Complete;
```

An exception inside the event does not escape to WebView2: it becomes a **500** response with the
message. To intercept another domain, register the filter first, for example
`FBrowser.AddWebResourceRequestedFilter('https://api.example.com/*')`. Full reference,
recipes and limitations in [docs/WebResourceRequested.md](docs/WebResourceRequested.md) ([Português](docs/WebResourceRequested_pt-BR.md)).

## Delphi ↔ JavaScript bridge

Simple message passing between the page and Delphi. For commands with Promises, errors and events,
build your own protocol on top of `OnWebMessageJson`.

From the page to Delphi:

```js
window.chrome.webview.postMessage('time');              // arrives as text
window.chrome.webview.postMessage({ action: 'save' });  // arrives as JSON
```

From Delphi to the page:

```pascal
FBrowser.PostWebMessage('Hello from Delphi');   // arrives in event.data
FBrowser.ExecuteScript('add(20, 22)');          // OnScriptResult receives '42'
```

```js
window.chrome.webview.addEventListener('message', function (e) { console.log(e.data); });
```

## Robustness

**Exceptions in events:** no exception escapes from a callback into WebView2. An exception in a
control event (`OnNavigationStarting`, `OnTitleChanged`...) goes to `OnCallbackError`;
without a handler, it goes to the application's default handler (the VCL error dialog). This way
WebView2 is not destabilized and navigation continues. In `OnWebResourceRequested`, the exception
becomes a 500 response.

**Process crash (`OnProcessFailed`):** WebView2 runs in separate processes that can crash.
The event receives the kind (`COREWEBVIEW2_PROCESS_FAILED_KIND_*`) and an action already filled in
with the default:

| Failure | Default action | Effect |
|---|---|---|
| Browser (`..._BROWSER_PROCESS_EXITED`) | `pfaRecreate` | Recreates the WebView (outside the callback) and returns to the last URL. Filters, scripts and options are reapplied |
| Renderer (`..._RENDER_PROCESS_EXITED`) | `pfaReload` | Reloads the page |
| Others (hang, GPU, utilities, frames) | `pfaNone` | Nothing: WebView2 recovers on its own, or the app decides |

```pascal
procedure TForm1.BrowserProcessFailed(Sender: TObject; Kind: Integer; var Action: TWV2ProcessFailedAction);
begin
  Log(Format('WebView2 process failed (kind %d)', [Kind]));
  if Kind = COREWEBVIEW2_PROCESS_FAILED_KIND_RENDER_PROCESS_UNRESPONSIVE then
    Action := pfaReload;  { also reloads when the page hangs }
end;
```

Do not open modal windows inside this (or any other) WebView2 event: post a message and
ask the user after the callback returns.

**Handle recreation:** changing the control's `Parent`, changing the form's `BorderStyle` and other
VCL operations destroy and recreate the window handle. The WebView2 window is a child of that
handle and would die with it. The control "parks" the WebView in a hidden window during the switch
and gives it back to the new handle, without reloading the page.

**Release:** when the control is destroyed (or on `Recreate`), the handlers are removed with their
tokens (`remove_*`), the controller is closed, and late callbacks from the old WebView are ignored.

## Tests

```bat
Tests\run-tests.bat
```

28 DUnit tests of the parts that do not need the runtime: intercepted request and response (with
a fake `ICoreWebView2WebResourceRequest`), URL and query decoding, MIME, resource names,
environment options and the control before `CreateWebView`. Requires DUnit (`DUNIT` variable,
default `C:\dev\lib\dunit\src`).

What depends on the runtime (process crash, handle recreation, deferred response, `Language` and
`AdditionalBrowserArguments`, callback protection) was verified with an integration app
using the DevTools protocol, including deliberately killing the renderer and the browser process.

## Delphi 7 pitfalls with WebView2

1. **BOOL is `-1`.** In Delphi, `LongBool(True)` is `-1`, but WebView2 compares against `TRUE` (1)
   and treats `-1` as false. In the first version, `put_IsScriptEnabled(True)` *turned off*
   JavaScript. Every **input** BOOL parameter uses the `WV2BOOL` type (Integer) and must receive
   `BoolToWV2(...)`. Output parameters can stay as `BOOL`.
2. **Floating-point exceptions.** Chromium expects these exceptions to be masked.
   `WebView2Browser.pas` calls `Set8087CW($133F)` at initialization; without it, random
   "Invalid floating point operation" errors appear.
3. **Callbacks without anonymous methods.** Each callback is a small `TInterfacedObject` class that
   forwards the call to the control. In the control's `Destroy` they are detached, so that a
   late callback does not touch an already freed object.
4. **vtable order.** In `WebView2.pas` the order of each interface's methods is the order of the
   vtable. Do not remove or reorder methods, even unused ones; they are there to occupy the slot.
5. **STA apartment.** `ComObj` is in the `uses` clause so that `Application.Initialize` calls
   `CoInitialize`.
6. **Strings.** Strings returned by WebView2 (`LPWSTR*`) must be freed with
   `CoTaskMemFree`; use `TakeCoTaskString`. The `LPCWSTR` received in `ExecuteScript`, however,
   belongs to WebView2: copy it, do not free it.

## Limitations

- Only a subset of the API is declared. To use another feature, declare the interface in
  `WebView2.pas`, checking the method order in `WebView2.h`.
- 32-bit only (a Delphi 7 limit).
- `Response.Complete` (deferred response) must be called on the UI thread.
- Deferred responses still pending when the WebView is released or recreated are discarded; do not
  call `Complete` on them after that.
