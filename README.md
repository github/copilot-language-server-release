# @github/copilot-language-server

The Copilot Language Server enables any editor or IDE to integrate with GitHub
Copilot via [the language server protocol](https://microsoft.github.io/language-server-protocol/).

**[GitHub Copilot](https://github.com/features/copilot)** is an AI pair programmer tool that helps you write code faster and smarter.

**Sign up for [GitHub Copilot Free](https://github.com/settings/copilot)!**

Please see [terms of use for GitHub Copilot](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#github-copilot)

## Getting Started

To integrate with the Copilot Language Server, download the latest release from npm:

```sh
npm install @github/copilot-language-server
```

To run the language server, platform-specific binaries are available as separate
packages included as optional dependencies. For example, [`@github/copilot-language-server-darwin-arm64`](https://www.npmjs.com/package/@github/copilot-language-server-darwin-arm64), for macOS on arm64:
example, for macOS on arm64:

```sh
./node_modules/@github/copilot-language-server-darwin-arm64/copilot-language-server --version
```

If repackaging the language server, all platform-specific binaries are available
in the releases: https://github.com/github/copilot-language-server-release/releases
For example, to download a zip of all of the binaries together:

```
gh release download -R github/copilot-language-server-release -p 'copilot-language-server-native-*'
```

Or you can use [Node.js](https://nodejs.org/en/download/) version 20.8 or later:

```sh
npx @github/copilot-language-server --version
```

```sh
node ./node_modules/@github/copilot-language-server/dist/language-server.js --version
```

If using the `language-server.js` distribution, it is necessary to retain the entire `dist` directory contents.

Communication with the language server typically happens over stdio with `--stdio`. The `language-server.js`
distribution additionally supports Node IPC with `--node-ipc`.

## Communication Protocol

The [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) is used to communicate with
the client. The base protocol is [JSON-RPC 2.0 with additional
headers](https://microsoft.github.io/language-server-protocol/specifications/specification-current/#baseProtocol).

The Copilot Language Server attempts to follow the LSP spec as closely as possible, but many custom messages are
employed to support the unique features of Copilot.

## Initialization

Per the LSP spec, the client is expected to send a
[`initialize`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize)
request to the language server as a first message. Example parameters:

```json
{
    "processId": 1234,
    "workspaceFolders": [
        {
            "uri": "file:///home/user/project"
        }
    ],
    "capabilities": {
        "workspace": {"workspaceFolders": true}
    },
    "initializationOptions": {
        "editorInfo": {
            "name": "GNU ed",
            "version": "1.19"
        },
        "editorPluginInfo": {
            "name": "GitHub Copilot for ed",
            "version": "1.0.0"
        }
    }
}
```

After receiving the response to `initialize`, the second message the client must send is an
[`initialized`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialized)
notification.

## Configuration Management

After initialization, clients should send a
[`workspace/didChangeConfiguration`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_didChangeConfiguration)
notification to inform the language server about the initial configuration, and again each time the configuration
changes. Example parameters:

```json
{
     "settings": {
        "http": {
            "proxy": "http://localhost:8888",
            "proxyStrictSSL": true,
            "proxyKerberosServicePrincipal": "spn"
        },
        "telemetry": {
            "telemetryLevel": "all"
        },
        "github-enterprise": {
            "uri": "https://example.ghe.com"
        }
    }
}
```

Pull based configuration on
[`workspace/configuration`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_configuration)
is also supported.

## Workspace Folders Synchronization

Workspace folders are optional but highly recommended as they greatly improve the quality of suggestions. See the LSP spec for
[`workspace/didChangeWorkspaceFolders`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_didChangeWorkspaceFolders).

## Text Document Synchronization

Per the LSP spec for
[text document synchronization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization),
support for `textDocument/didOpen`, `textDocument/didChange`, and `textDocument/didClose` is required, using incremental
sync.

## Text Document Focusing

Additionally a custom `textDocument/didFocus` notification should be sent when the client focuses another document
(e.g., changing tabs). Example parameters:

```json
{
    "textDocument": {
        "uri": "file:///path/to/file"
    }
}
```

If no document is focused, a request with no parameters (`{}`) may be sent.

## Status Notification

The status is sent to the client as a `didChangeStatus` notification. Typically this would be conveyed to the user in a
status bar icon or other UI affordance.

Notification includes the following fields:

- `message` - a string describing the status (can be empty)
- `kind` - status of the language server:
   - `'Normal'` - When everything is working normally.
   - `'Error'` - When not authorized and authenticated.
   - `'Warning'` - When there is a temporary issue, such as a failed HTTP request.
   - `'Inactive'` - When the current file is ignored due to file size or content exclusions.

## Authentication

To sign in, call `signIn`. This will return a response like the following:

```json
{
    "userCode": "ABCD-EFGH",
    "command": {
        "command": "github.copilot.finishDeviceFlow",
        "arguments": [],
        "title": "Sign in"
    }
}
```

Instruct the user to copy the `userCode` to their clipboard (and/or copy it programmatically). When the user is ready, invoke
[`workspace/executeCommand`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_executeCommand)
to execute the `command`. This will open a browser window that walks the user through the authentication process.
Shortly (up to 10 seconds) after the user finishes signing in, you should see a status notification reflecting the new
state.

If available, the language server will use
[`window/showDocument`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#window_showDocument)
to open the URL. Otherwise, it will attempt to open the URL natively (e.g., with `/usr/bin/open` on macOS).

To sign out, call `signOut`.

## Inline Completions

The [`textDocument/inlineCompletion`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.18/specification/#textDocument_inlineCompletion)
request from the draft version of the next LSP spec is used to retrieve inline ("ghost text") completions, with some
non-standard additions (`textDocument.version` and `formattingOptions`) to the parameters:

```json
{
    "textDocument": {
        "uri": "file:///path/to/file",
        "version": 0
    },
    "position": {"line": 1, "character": 2},
    "context": {
        "triggerKind": 2
    },
    "formattingOptions": {
        "tabSize": 4,
        "insertSpaces": true
    }
}
```

The result is an object containing an `items` array.

```json
{
    "items": [
        {
            "insertText": "a helpful suggestion",
            "range": {
                "start": {"line": 1, "character": 0},
                "end": {"line": 1, "character": 2}
            },
            "command": {
                "command": "github.copilot.didAcceptCompletionItem",
                "arguments": ["some id"]
            }
        }
    ]
}
```

Newlines in `insertText` should be normalized as is appropriate for the editor before inserting into the document.

The `command` field, per the LSP spec, is called via
[`workspace/executeCommand`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_executeCommand)
*after* the user accepts the completion. Copilot uses this for acceptance telemetry.

The LSP spec does not provide an event for showing the completion, so a custom `textDocument/didShowCompletion` is used.
Call it with an `item` parameter containing the full completion item:

```json
{
    "item": {
        "insertText": "a helpful suggestion",
        "range": {
            "start": {"line": 1, "character": 0},
            "end": {"line": 1, "character": 2}
        },
        "command": {
            "command": "github.copilot.didAcceptCompletionItem",
            "arguments": ["some id"]
        }
    }
}
```

Similarly, for partial acceptance, send a `textDocument/didPartiallyAcceptCompletion` notification with the full
item, plus the length (in UTF-16 codepoints) of the completion that was accepted:

```json
{
    "item": {
        "insertText": "a helpful suggestion",
        "range": {
            "start": {"line": 1, "character": 0},
            "end": {"line": 1, "character": 2}
        },
        "command": {
            "command": "github.copilot.didAcceptCompletionItem",
            "arguments": ["some id"]
        }
    },
    "acceptedLength": 9
}
```

Note that the `acceptedLength` includes everything from the start of `insertText` to the end of the accepted text. It is
*not* the length of the accepted text itself.

## Panel Completions

Panel completions are used for "Open Copilot" style completions. They are similar to inline completions, but are shown
in a separate panel. They are retrieved using the custom `textDocument/copilotPanelCompletion` request, which has
parameters closely modeled after `textDocument/inlineCompletion`:

```json
{
    "textDocument": {
        "uri": "file:///path/to/file",
        "version": 0
    },
    "position": {"line": 1, "character": 2},
    "partialResultToken": "some token"
}
```

If provided, the `partialResultToken` is used to stream
[partial results](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#partialResults)
back to the client. Both the return type and the partial result type are the same as for inline completions: an object
containing an `items` array. These items have the same `command` that must be invoked with `workspace/executeCommand`
after accepting to trigger acceptance telemetry. Partial acceptance and shown events are not supported here.

## Cancellation

`textDocument/inlineCompletion` supports cancel-previous strategy for cancellation: If you send new completions
request, the previous request is cancelled.

Additionally the LSP `$/cancelRequest` method is supported to cancel any request. It is strongly encouraged to eagerly
cancel completion requests as soon as possible.

## Logs

Logs are sent to the client as
[`window/logMessage`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#window_logMessage)
notifications.

## Messages

The client may be sent
[`window/showMessageRequest`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#window_showMessage)
requests. Support for these messages is essential as they are used for important notifications including account and
billing.


---

## License

Distributed under the MIT license. See LICENSE for details.
