# diff-sync-js

A JavaScript implementation of Neil Fraser Differential Synchronization Algorithm

Diff Sync Writing: https://neil.fraser.name/writing/sync/

## Use Case

Differential synchronization algorithm keep two or more copies of the same document synchronized with each other in real-time. The algorithm offers scalability, fault-tolerance, and responsive collaborative editing across an unreliable network.

## Demo

http://wztechs.com/diff-sync-text-editor/demo/client/

## How to install

When use npm
`npm install diff-sync-js`

When use html
`<script src="./dist/diffSync.js"></script>`

## Dependencies

`json-fast-patch`

## To Test

`npm run test`

## How to use

1. Initial diffSync instance

```
    var diffSync = new DiffSyncAlghorithm({
        jsonpatch: jsonpatch,
        thisVersion: "m",
        senderVersion: "n",
        useBackup: true,
        debug: true
    });
```

2. Initialize container

```
    var container = {};
    diffSync.initObject(container, mainText);
```

3. When Send Payload

```
    diffSync.onSend({
        container,
        mainText,
        whenSend(senderVersion, edits) {
            send({
                type: "PATCH",
                {
                    senderVersion,
                    edits
                }
            });
        }
    });
```

4. When Receive Payload

```
    diffSync.onReceive({
        payload,
        container,
        onUpdateMain(patches, operations) {
            mainText = jsonpatch.applyPatch(mainText, operations).newDocument;
        },
        afterUpdate(senderVersion) {
            send({
                type: "ACK",
                payload: {
                    senderVersion
                }
            });
        }
    });
```

5. When Receive Ack

```
    diffSync.onAck(container, payload);
```

## API

constructor

```
     /**
     * @param {object} options.jsonpatch json-fast-patch library instance (REQUIRED)
     * @param {string} options.thisVersion version tag of the receiving end
     * @param {string} options.senderVersion version tag of the sending end
     * @param {boolean} options.useBackup indicate if use backup copy (DEFAULT true)
     * @param {boolean} options.debug indicate if print out debug message (DEFAULT false)
     */
    constructor({ jsonpatch, thisVersion, senderVersion, useBackup = true, debug = false })

```

initObject

```
    /**
     * Initialize the container
     * @param {object} container any
     * @param {object} mainText any
     */
    initObject(container, mainText)
```

onReceive

```
    /**
     * On Receive Packet
     * @param {object} options.payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     * @param {object} options.container container object {shadow, backup}
     * @param {func} options.onUpdateMain (patches, patchOperations, shadow[thisVersion]) => void
     * @param {func} options.afterUpdate (shadow[senderVersion]) => void
     * @param {func} options.onUpdateShadow (shadow, patch) => newShadowValue
     */
    onReceive({ payload, container, onUpdateMain, afterUpdate, onUpdateShadow })
```

onSend

```
    /**
     * On Sending Packet
     * @param {object} options.container container object {shadow, backup}
     * @param {object} options.mainText any
     * @param {func} options.whenSend (shadow[senderVersion], shadow.edits) => void
     * @param {func} options.whenUnchange (shadow[senderVersion]) => void
     */
    onSend({ container, mainText, whenSend, whenUnchange })

```

onAck

```
     /**
     * Acknowledge the other side when no change were made
     * @param {object} container container object {shadow, backup}
     * @param {object} payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     */
    onAck(container, payload)
```

onAck

```
    /**
     * Acknowledge the other side when no change were made
     * @param container container object {shadow, backup}
     * @param payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     */
    onAck(container, payload)
```

clearOldEdits

```
    /**
     * clear old edits
     * @param {object} shadow
     * @param {string} version
     */
    clearOldEdits(shadow, version)
```

strPatch

```
    /**
     * apply patch to string
     * @param {string} val
     * @param {patch} patch
     * @return string
     */
    strPatch(val, patch)
```
## School work

### Which implementation of diff-patch algorithms are used? Are there better/newer ones?

This implementation uses the fast-json-patch package, version 3.x.x.
It uses the RFC6902 (src: https://www.npmjs.com/package/fast-json-patch).

We found the following:
- https://github.com/benjamine/jsondiffpatch (uses LCS and the google one)
- https://github.com/cujojs/jiff (also uses RFC6902)
- https://github.com/google/diff-match-patch (uses Myer's diff algorithm)

### Where are the documents and shadows? How is a document copied?

They are saved in a container object, which is created in the client or server.
The DS implementation uses a parameter to get the reference to this object.
The document and shadow in a container is accessible with:

```js
container.shadow
// or
container.backup
```

A (json) document is copied with:

```js
// it uses the fast-json-patch library
jsonpatch.deepClone(mainText);
```

### How and Why can we adjust the sync-cycle? What are the dis-/advantages?

**Server Receive Chance = 0%:**  
No data can be received anymore. When the page is reloaded, the changes made **before** SRC=0% are displayed.  
If SRC is then increased and bits are changed, another update must be made for the changes to fully appear on the other side (the updated version).

---

**Client Receive Chance = 60%:**  
If the Client Receive Chance is set to 60%, some bits may not be received. For example:  
*(User tests with "Hallo" remain unchanged.)*  
Here, a change to the message is made, but not all bits are transmitted to the other side. Adding another bit ensures that both the previous and current updates are transmitted, resulting in the fully updated message being displayed.

---

**Server + Client Receive Chance = 50%:**  
When both Server and Client Receive Chance are set to 50%, the error margin increases significantly. For example:  
*(User tests remain unchanged.)*

In this case, the error is larger, and the data only updates fully once an individual bit is successfully transmitted.

If both sides make changes **simultaneously**, the fields can show different texts. For example:  
*(User tests with "Hallo" remain unchanged.)*

When a bit is successfully received, the new bits from one user are appended **before** the changes made by the other user that were not transmitted.

The disadvantage might be, that if both the Client AND the Server Receive Chances are low, there might or will be a colission and the Text will not be the same. The advantage would be that, if one of the two Receive Chances are low, it's not an issue, since all of the changes will be updated once a bit is received again!

### Where / how is the edit stack implemented? How can the stack be packaged for sending? 

It is implemented as JS array, in the container:

```js
container.shadow.edits
```

Because the stack is an array of json objects, it can easily be stringified:

```js
socket.send(JSON.stringify(data));
```

### Is it possible to deploy a Peer-to-Peer version?

### How is it possible to use the API in other JS projects?

### Are the JSON documents interchangeable with other kind of documents?

It would require refactoring of the code since the fast-json-patch library is extensively used in various locations,
but in theory, yes.

### How are conflicts handled?

