# Frequently Asked Questions

This is a list of questions we frequently get and problems that are often encountered. 
Because this project is still in development, we have missing pieces that are commonly
encountered and prefer not to get new issues filed in our bug tracker.

* [Why Swift?](#why-swift)
* [Why do I get "error: array input is not a constant array of tensors"?](#why-do-i-get-error-array-input-is-not-a-constant-array-of-tensors)
* [Why do I get "error: internal error generating TensorFlow graph: GraphGen cannot lower a 'send/receive' to the host yet"](#why-do-i-get-error-internal-error-generating-tensorflow-graph-graphgen-cannot-lower-a-sendreceive-to-the-host-yet)
* [How can I use Python 3 with the Python module?](#how-can-i-use-python-3-with-the-python-module)

## Why Swift?

The short answer is that our decision was driven by the needs of the core [Graph Program
Extraction](docs/GraphProgramExtraction.md) compiler transformation that started the whole
project.  We have a long document that explains all of this rationale in depth named "[Why 
*Swift* for TensorFlow?](docs/WhySwiftForTensorFlow.md)".

Separate from that, Swift really is a great fit for our purposes, and a very nice language.

## Why do I get ["error: array input is not a constant array of tensors"](https://github.com/tensorflow/swift/issues/10)?

If you ran into this error, you likely wrote some code using `Tensor` without running Swift with optimizations (`-O`). 

The `-O` flag enables optimizations and is currently required for the [graph program extraction
algorithm](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md) to work correctly.
We're working on making `-O` not required, but in the meantime you need to specify it.

Here's how to enable optimizations in different environments:

* REPL: No need to add extra flags. Optimizations are on by default. 
* Interpreter: `swift -O main.swift`
* Compiler: `swiftc -O main.swift`
* `swift build`: `swift build -Xswiftc -O`
* Xcode: Go to `Build Settings > Swift Compiler > Code Generation > Optimization Level` and select `Optimize for Speed [-O]`.
  * You may also need to add `libtensorflow.so` and `libtensorflow_framework.so` to `Linked Frameworks and Libraries` and change `Runtime Search Paths`.
    See [this comment](https://github.com/tensorflow/swift/issues/10#issuecomment-385167803) for specific instructions with screenshots.

## Why do I get ["error: internal error generating TensorFlow graph: GraphGen cannot lower a 'send/receive' to the host yet"](https://github.com/tensorflow/swift/issues/8)?

### Context

In Swift for Tensorflow, a Swift program is executed between the "host" (Swift binary) and the "device" (TensorFlow).
During program compilation, the compiler finds all of the `Tensor`-related code, extracts it and builds a TensorFlow graph to be run on the device. 

The compiler tries to inline as much tensor related code into one "unit of Graph Program Extraction (GPE)" as possible. This is for performance reason -- a larger TensorFlow graph is expected to be more efficient for execution.

In some cases, the extracted tensor computation has some interleaving host logic, in which case the compiler adds "send" or "receive" nodes in the TensorFlow graph for tensor communication between host and device.

Send/receive aren't fully implemented, which explains why you got your error. The core team is working on it as a high-priority feature. Once send/receive are done, this error should go away!

For more context, please read about the [graph program extraction algorithm](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md),
in particular about [host-graph communication](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md#adding-hostgraph-communication).

### Example 1

```swift
import TensorFlow
var x = Tensor([[1, 2], [3, 4]])
print(x)
x = x + 1
```
In this case, the `Tensor` code to-be-extracted are lines 2 and 4 (the calculations and assign/update to `x`).

However, notice line 3: it's a call to the `print` function (which must be run on the host), and it requires the value of `x` (which is computed on the device). It's also not the last computation run in the graph (which is line 4).

As such, the compiler adds a "send" node in the TensorFlow graph to send a copy of the initial `x` to the host for printing.

To work around the generated "send" for this particular example, you can reorder the code:

```swift
import TensorFlow
var x = Tensor([[1, 2], [3, 4]])
x = x + 1
print(x)
```

The `Tensor` code is no longer "interrupted" by host code so there's no need for "send".

### Example 2

Say you top level code (i.e., the implicit Swift main() function) has the following, where foo() and bar() are functions returning some tensor.
```swift
print(foo())
print(bar())
```

The code is equivalent to:
```swift
let x = foo()
print(x)
let y = bar()
print(y)
```
`print(x)` above interleaves with the tensor logic in `foo()` and `bar()`, resulting in tensor sends.

One work-around as mentioned above is to refactor the code and bring the tensor code together (e.g. move print(x) downward).

Another is to mark `foo()` and `bar()` non-inlineable. e.g.
```swift
@inline(never)
public func bar(_ x: Double) -> Tensor<Double> {
  return foo(x)
}
```

This will prevent the body of bar() from being inlined into the body of main(), when compiler processes main(). This will cause compiler to generate separate TF graphs for `foo()` and `bar()`.

**Note**: Given our implementation is going to be moving a lot and coming together in the next couple months, this is the most pragmatic workaround.  It is a pain, but to get the most predictability (without knowing all of the ins and outs of how things work), the best way to go is to define functions that do all the tensor computation inside of them (and mark them @inline(never) and public just to be safe), and do no host communication.  This will ensure that all the host values become arguments and results of the generated tensor program.

## How can I use Python 3 with the `Python` module?

Currently, Swift is hard-coded to use Python 2.7.
Adding proper Python 3 support is non-trivial but in discussion.
See [this issue](https://github.com/tensorflow/swift/issues/13) for more information.