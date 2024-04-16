# `jest-chrome`

A complete mock of the Chrome API for Chrome extensions, for use
with Jest.

TypeScript support is built in. Each function and event is based
on the
[`@types/chrome`](https://www.npmjs.com/package/@types/chrome)
package.

## Installation

```sh
npm i jest-chrome -D
```

Set `chrome` in the global scope during setup so that it is
mocked in imported modules. Add a setup file to `jest.config.js`:

```javascript
// jest.config.js

module.exports = {
   // Add this line to your Jest config
   setupFilesAfterEnv: ['./jest.setup.js'],
}
```

Use the setup file to assign the mocked `chrome` object to the
`global` object:

```javascript
// jest.setup.js

Object.assign(global, require('jest-chrome'))
```

Import `chrome` from `jest-chrome` for Intellisense and linting.
This is the same object as `chrome` in the global scope.

```javascript
import { chrome } from 'jest-chrome'
```

## Usage

> All of the following code blocks come from
> [`tests/demo.test.ts`](tests/demo.test.ts).

### Events

Each mocked Event has all the normal methods (`addListener`,
`hasListener`, `hasListeners`, and `removeListener`) plus two
more: `callListeners` and `clearListeners`.

`callListeners` triggers a specific Event. Call `callListeners`
with the arguments you expect Chrome to pass to your event
listeners. Each event listener for that Event will be called with
those arguments.

`clearListeners` removes all listeners for a specific Event.

```javascript
test('chrome api events', () => {
   const listenerSpy = jest.fn()
   const sendResponseSpy = jest.fn()

   chrome.runtime.onMessage.addListener(listenerSpy)

   expect(listenerSpy).not.toBeCalled()
   expect(chrome.runtime.onMessage.hasListeners()).toBe(true)

   chrome.runtime.onMessage.callListeners(
           { greeting: 'hello' }, // message
           {}, // MessageSender object
           sendResponseSpy, // SendResponse function
   )

   expect(listenerSpy).toBeCalledWith(
           { greeting: 'hello' },
           {},
           sendResponseSpy,
   )
   expect(sendResponseSpy).not.toBeCalled()
})
```

### Synchronous functions

Some Chrome API functions are synchronous. Use these like any
mocked function:

```javascript
test('chrome api functions', () => {
   const manifest = {
      name: 'my chrome extension',
      manifest_version: 2,
      version: '1.0.0',
   }

   chrome.runtime.getManifest.mockImplementation(() => manifest)

   expect(chrome.runtime.getManifest()).toEqual(manifest)
   expect(chrome.runtime.getManifest).toBeCalled()
})
```

### Functions with callbacks

Most Chrome API functions do something asynchronous. They use
callbacks to handle the result. The mock implementation should be
set to handle the callback.

> Mocked functions have no default mock implementation!

```javascript
test('chrome api functions with callback', () => {
   const message = { greeting: 'hello?' }
   const response = { greeting: 'here I am' }
   const callbackSpy = jest.fn()

   chrome.runtime.sendMessage.mockImplementation(
           (message, callback) => {
              callback(response)
           },
   )

   chrome.runtime.sendMessage(message, callbackSpy)

   expect(chrome.runtime.sendMessage).toBeCalledWith(
           message,
           callbackSpy,
   )
   expect(callbackSpy).toBeCalledWith(response)
})
```

### Callbacks and `chrome.runtime.lastError`

When something goes wrong in a callback, Chrome sets
`chrome.runtime.lastError` to an object with a message property.
If you need to test this, set and clear `lastError` in the mock
implementation.

> Remember that `lastError` is always undefined outside of a
> callback!

`lastError` is an object with a
[getter function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get)
for the `message` property. If `message` is not checked, Chrome
will log the error to the console. To emulate this, simply set
`lastError` to an object with a getter that wraps a mock, as seen
below:

```javascript
test('chrome api functions with lastError', () => {
   const message = { greeting: 'hello?' }
   const response = { greeting: 'here I am' }

   // lastError setup
   const lastErrorMessage = 'this is an error'
   const lastErrorGetter = jest.fn(() => lastErrorMessage)
   const lastError = {
      get message() {
         return lastErrorGetter()
      },
   }

   // mock implementation
   chrome.runtime.sendMessage.mockImplementation(
           (message, callback) => {
              chrome.runtime.lastError = lastError

              callback(response)

              // lastError is undefined outside of a callback
              delete chrome.runtime.lastError
           },
   )

   // callback implementation
   const lastErrorSpy = jest.fn()
   const callbackSpy = jest.fn(() => {
      if (chrome.runtime.lastError) {
         lastErrorSpy(chrome.runtime.lastError.message)
      }
   })

   // send a message
   chrome.runtime.sendMessage(message, callbackSpy)

   expect(callbackSpy).toBeCalledWith(response)
   expect(lastErrorGetter).toBeCalled()
   expect(lastErrorSpy).toBeCalledWith(lastErrorMessage)

   // lastError has been cleared
   expect(chrome.runtime.lastError).toBeUndefined()
})
```

### Contributions

The `chrome` object is based on schemas from the Chromium
project

#### How to extend the schema

1. #### Top-Middle level objects (E.g. add `local` to `chrome.storage`).
   Let's add `local` schema to `chrome.storage`. If you want to add some non-end property then you should add it as an object key
    ```json
     {
         "storage": {
            // ... 
            "local": {
               "type": "property",
               "name": "local",
               "value": "%storage%"
            }    
         }
      }
    ```
2. #### End level objects
   There are 3 types of end level entities: property, function and event. All of them has 2 same properties
   `type: 'function' | 'event' | 'property'` and `name: string`.
   Property has one additional field called value (`value: any`). Value represents the actual value of the property
   except 2 cases. When value equals `%storage%` or `%chromeSetting%` it will be replaced with the:
```typescript
// %storage%
{
   clear: jest.fn(),
   get: jest.fn(),
   getBytesInUse: jest.fn(),
   setAccessLevel: jest.fn(),
   remove: jest.fn(),
   set: jest.fn(),
   onChanged: createEvent((...args: any[]) => args)
}
// %chromeSetting%
{
   clear: jest.fn(),
   get: jest.fn(),
   onChange: createEvent((...args: any[]) => args),
   set: jest.fn()
}
```
Functions and events has their own property called parameters, which is representing the parameters of each:
```typescript
export interface Parameter {
   /**
    * Represents the name of the parameter
    */
   name: string
   /**
    * Represents if parameter optional or not
    */
   optional: boolean
   /**
    * Don't have a clue what it does =)
    */
   parameters: number
   /**
    * Type of the parameter
    */
   type: string
}
```
Example of adding end level properties
```json
{
   "storage": {
      // ... 
      "local": {
         "type": "property",
         "name": "local",
         "value": "%storage%"
      }
   },
   "tabs": {
      // ... 
      "connect": {
         "type": "function",
         "name": "connect",
         "parameters": [
            {
               "name": "tabId",
               "optional": false,
               "length": 0,
               "type": "integer"
            },
            {
               "name": "connectInfo",
               "optional": true,
               "length": 0,
               "type": "object"
            }
         ]
      }
   },
   "webNavigation": {
      "onBeforeNavigate": {
         "type": "event",
         "name": "onBeforeNavigate",
         "parameters": [
            {
               "name": "details",
               "optional": false,
               "parameters": 0,
               "type": "object"
            }
         ],
         "rules": false
      }
   }
}
```