![Scala CI](https://github.com/cjermain/flink-onnx-pytorch/workflows/Scala%20CI/badge.svg)

# PyTorch on Flink with ONNX

This repository shows an example of using [Apache Flink][1] to serve an [ONNX model][2]
that was created by [PyTorch][3]. While the initial model is simple, it shows the
process by which a machine learning model can be created in Python, and then
served by a Scala program written for streaming with Flink.

[1]: https://flink.apache.org/
[2]: https://onnx.ai/
[3]: https://pytorch.org/

## Training in PyTorch

The [`training/`](./training) directory provides two example Jupyter Notebooks
that uses PyTorch to create simple networks. The networks are exported to ONNX
through PyTorch. The simple network takes a variable length floating point tensor
and adds a constant offset. The MNIST network solves the classic handwritten number
classification problem that is a mainstay of neural network tutorials.

## Packaging the JAR

The ONNX models are [packaged as a resource](./src/main/resources/), so that they can be
distributed to the Flink TaskManagers in the same JAR as the code.

The Flink application follows the standard template for SBT. The JAR can be built
using `sbt` as follows:

```bash
sbt clean assembly
```

The Scala code is also tested to ensure quality, and those tests are run by `sbt`.
These tests exercise the inference of the ONNX models.

The contents of the JAR can be inspected to see the included ONNX models and the
[`onnxruntime`][4] library.

```bash
jar tf ./target/scala-2.11/flink-onnx-pytorch-assembly-0.0.1.jar
```

## Serving with onnxruntime

The ONNX model is served using the [`onnxruntime`][4] library. The [Java API][5]
is used, which provides a similar interface to Python. The `OrtModel` class is added as a
convenience wrapper to handle loading from the resources directory.

Running the model inference for the simple case is wrapped by the `AddFive` class,
which extends the [`RichMapFunction`][6]. This allows the `open` and `close` methods
to handle the model loading and closing. The `map` method runs the input value through
the ONNX model.

The types need to be handled properly, since fractional values default to `Double`
in Scala unless specifically identified.

[4]: https://github.com/microsoft/onnxruntime
[5]: https://github.com/microsoft/onnxruntime/tree/master/java
[6]: https://ci.apache.org/projects/flink/flink-docs-release-1.12/api/java/org/apache/flink/api/common/functions/RichMapFunction.html

## Streaming in Flink

With the JAR built, the Flink jobs can be submitted. The jobs are written over a static
dataset of numbers for the simple example. This could be easily replaced by another source.
The jobs output the results to stdout for simplicity, which could also be replaced by
another sink. The [DataStream API][7] is used in this example.

To run the JAR on a local cluster, make sure to start it first.

```bash
./flink-1.13.0/bin/start-cluster.sh
```

The `com.datacolin.Job` can be submitted to the cluster through the CLI:

```bash
./flink-1.13.0/bin/flink run -c com.datacolin.Job ./flink-onnx-pytorch/target/scala-2.11/flink-onnx-pytorch-assembly-0.0.1.jar
```

The local cluster will provide output in the `log/` directory, which can be watched:

```bash
tail -f ./flink-1.13.0/log/*.out
```

The static dataset should be output with the added offset.

The MNIST example can be run using the `ai.vectra.Job`.

[7]: https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/datastream_api.html

## Future directions

This demonstration provides the basic working pieces to get a Python machine learning
model running in a streaming Scala application in Flink. There are a number of
interesting real-time applications to try next.
