# `BufferBackedObject`

**`BufferBackedObject` creates objects that are backed by an `ArrayBuffer`**. It takes a schema definition and de/serializes data on-demand using [`DataView`][dataview] under the hood. The goal is to make [`ArrayBuffer`][arraybuffer]s more convenient to use.

```
npm i -S buffer-backed-object
```

## Why?

### Web Workers

When using [Web Workers], the performance of `postMessage()` (or the [structured clone algorithm][structured clone] to be exact) is often a concern. While [`postMessage()` is a lot faster than most people give it credit for][is postmessage slow], it can still occasionally be a bottle-neck, especially with bigger payloads. [`ArrayBuffer`][arraybuffer] and their [views][arraybufferview] are incredibly quick to clone (or can even be [transferred][transferable]), but getting data in and out of `ArrayBuffer`s can be cumbersome. `BufferBackedObject` makes this easy by giving you a (seemingly) normal JavaScript object that reads and write values from the `ArrayBuffer` on demand. This means that the serialization & deserialization costs are deferred to the point of access rather than paid upfront, as it is the case with `postMessage()`.

### WebGL

[WebGL Buffers][webgl buffer] can store multiple attributes per vertex using [`vertexAttribPointer()`][vertexattribpointer]. These attributes can be a 3D position, but also other additional data like a normal vector, a color or a texture ID. The underlying buffer contains all the attributes for all the vertices in an interleaved format, which can make manipulating that data quite hard. With `ArrayOfBufferBackedObjects` you can manipulate each vertex individually. Additionally, `ArrayOfBufferBackedObjects` is populated lazily (see more below), allowing you to handle big arrays of vertices more efficiently.

## Example

```js
import { BufferBackedObject } from "buffer-backed-object";

const buffer = new ArrayBuffer(100);
const view = new BufferBackedObject(buffer, {
  id: BufferBackedObject.Uint16({ endianness: "big" }),
  position: BufferBackedObject.NestedBufferBackedObject({
    x: BufferBackedObject.Float32(),
    y: BufferBackedObject.Float32(),
    z: BufferBackedObject.Float32(),
  }),
  normal: BufferBackedObject.NestedBufferBackedObject({
    x: BufferBackedObject.Float32(),
    y: BufferBackedObject.Float32(),
    z: BufferBackedObject.Float32(),
  }),
  textureId: BufferBackedObject.Uint8(),
});

view.id = 3;
console.log(new Uint8Array(buffer));
// logs: Uint8Array(100) [3, 0, ...]
console.log(JSON.stringify(view));
// logs: {"id": 3, "position": {"x": 0, ...}, ...}
```

`ArrayOfBufferBackedObjects` interprets the given `ArrayBuffer` as an _array_ of objects with the given schema:

```js
import {
  ArrayOfBufferBackedObjects,
  BufferBackedObject,
} from "buffer-backed-object";

const buffer = new ArrayBuffer(100);
const view = new ArrayOfBufferBackedObjects(buffer, {
  id: BufferBackedObject.Uint16({ endianness: "big" }),
  position: BufferBackedObject.NestedBufferBackedObject({
    x: BufferBackedObject.Float32(),
    y: BufferBackedObject.Float32(),
    z: BufferBackedObject.Float32(),
  }),
  normal: BufferBackedObject.NestedBufferBackedObject({
    x: BufferBackedObject.Float32(),
    y: BufferBackedObject.Float32(),
    z: BufferBackedObject.Float32(),
  }),
  textureId: BufferBackedObject.Uint8(),
});

// The struct takes up a total of 27 bytes, so
// 3 structs can fit into a 100 byte `ArrayBuffer`.
console.log(view.length);
// logs: 3

view[0].id = 1000;
view[1].id = 1001;
view[2].id = 1002;
console.log(new Uint8Array(buffer));
// logs: Uint8Array(100) [232, 3, ...]
console.log(JSON.stringify(view));
// logs: [{"id": 1000, ...}, {"id": 1001}, ...]
```

## API

The module has the following exports:

### `new BufferBackedObject(buffer, descriptors, {byteOffset = 0})`

The key/value pairs in the `descriptors` object must be declared in the same order as they are laid out in the buffer. The returned object has getters and setters for each of `descriptors` properties and de/serializes them `buffer`, starting at the given `byteOffset`.

The following descriptor types are available as built-ins:

- `BufferBackedObject.reserved(numBytes)`: A number of unused bytes. This field will now show up in the object.
- `BufferBackedObject.Int8()`: An 8-bit signed integer
- `BufferBackedObject.Uint8()`: An 8-bit unsigned integer
- `BufferBackedObject.Int16({endianness = 'little'})`: An 16-bit signed integer
- `BufferBackedObject.Uint16({endianness = 'little'})`: An 16-bit unsigned integer
- `BufferBackedObject.Int32({endianness = 'little'})`: An 32-bit signed integer
- `BufferBackedObject.Uint32({endianness = 'little'})`: An 32-bit unsigned integer
- `BufferBackedObject.BigInt64({endianness = 'little'})`: An 64-bit signed [`BigInt`][bigint]
- `BufferBackedObject.BigUint64({endianness = 'little'})`: An 64-bit unsigned [`BigInt`][bigint]
- `BufferBackedObject.Float32({endianness = 'little'})`: An 32-bit IEEE754 float
- `BufferBackedObject.Float64({endianness = 'little'})`: An 64-bit IEEE754 float (“double”)
- `BufferBackedObject.UTF8String(maxBytes)`: A UTF-8 encoded string with the given maximum number of bytes. Trailing NULL bytes will be trimmed after decoding.
- `BufferBackedObject.ArrayBuffer(size)`: An `ArrayBuffer` of the given size
- `BufferBackedObject.NestedBufferBackedObject(descriptors)`: A nested `BufferBackedObject` with the given descriptors
- `BufferBackedObject.NestedArrayOfBufferBackedObjects(numItems, descriptors)`: A nested `ArrayOfBufferBackedObjects` of length `numItems` with the given descriptors

### `new ArrayOfBufferBackedObjects(buffer, descriptors, {byteOffset = 0, length = 0})`

Like `BufferBackedObject`, but returns an _array_ of `BufferBackedObject`s. If `length` is 0, as much of the buffer is used as possible. The array is populated lazily under the hood for performance purposes. That is, the individual `BufferBackedObject`s will only be created when their index is accessed.

### `structSize(descriptors)`

Returns the number of bytes required to store a value with the schema outlined by `descriptors`.

## Defining your own descriptor types

All the descriptor functions return an object with the following structure:

```js
{
  size: 4, // Size required by the type
  get(dataView, byteOffset) {
    // Decode the value at byteOffset using
    // `dataView` or `dataView.buffer` and
    // return it.
  },
  set(dataView, byteOffset, value) {
    // Store `value` at `byteOffset` using
    // `dataView` or `dataView.buffer`.
  }
}
```

---

License Apache-2.0

[dataview]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView
[arraybuffer]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[web workers]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API
[structured clone]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm
[is postmessage slow]: https://surma.dev/things/is-postmessage-slow/
[arraybufferview]: https://developer.mozilla.org/en-US/docs/Web/API/ArrayBufferView
[transferable]: https://developer.mozilla.org/en-US/docs/Web/API/Transferable
[bigint]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt
[webgl buffer]: https://developer.mozilla.org/en-US/docs/Web/API/WebGLBuffer
[vertexattribpointer]: https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/vertexAttribPointer
