# fpgaconvnet on Maxeler

Running convolutional neural network (CNN) inference on maxeler dataflow engines (DFE).
This toolchain targets _multiple FPGAs_ on a single DFE device, where communication
between FPGAs is done primarily via maxring (sometimes using off-chip RAM).

This tool (as of writing) performs only feature extraction, namely the convolution,
pooling layers etc.

To use this tool, you need to provide a protobuf descriptor of the CNN you wish
to accelerate, and the tool will generate FPGA bistreams (ie: maxfiles).
*This means that you need to recompile multiple FPGA bistreams everytime you want to
run inference on a new neuralnetwork, or the available resources change.* This is
certainly not what you want for developing neural networks / testing them, but probably
something you want in a production setting for long-term usage (eg: classifying images
over the course of at least weeks).

The tool targets a high throughput + low-powered usage, that is, we aim to
have a lower Ops/Watt compared to conventional using GPUs.

The supported DFE cards include:

- MAIA

The design methodology and parameter optimisation will be uploaded soon,
along with a guide on how to do some hacking on this.

## Features and Non-Features

These are special features (and non-features) that are worth mentioning,
that might be foreign to deep-learning practitioners (or even experts
in the FPGA deep learning aceeleration field).

### What works

- *Automatic Design space exploration (DSE)* - automatically search for
  good loop unrolling parameters using a theoretical performance model.
- *Performance Estimation* - estimates the throughput of the design
  without performing a full-on compilation / simulation.
- *Compilation relaxation* - in case the compilation fails, the user
  can relax the resource utilisation to allow the design to fit on the
  FPGA more easily during the MPPR stage. This can be specified in the
  [convnet protobuf](protos/fpgaconvnet/protos/parameters.proto)
- *Simulating maxring connections without reconfiguration* When simulating
  a multi FPGA design WITHOUT reconfiguration, the maxring connections are
  mocked as PCIe connections to the CPU. This is useful for correctness
  verification.
- *Custom Fixed Point Precision and FPGA Frequency* Instead of using a
  single setting for all neural networks, the user can specify custom
  fixed point precisions (so long that it sums up to 16 bits) and FPGA
  frequencies for each neural network.

### What doesn't Work

- *DSE on pipelined FPGAs with Reconfiguration* - This is currently
  left as a future work of the project, due to the inherent difficulty
  in optimisation.
- *Simulation with reconfiguration* - As the maxcompiler resets the LMem
  when a new maxfile is loaded, the temporal results that is written to
  LMem is wipped. This makes it impossible to simulate design with
  runtime FPGA reconfiguration.
- *Simulation of extremely large neural networks* While this should
  theoretically work, large neural networks take a long time to simulate.
  The time required long exceeds the number maxeler timeout of 30 minutes
  for even a single image on Alexnet.
- *Fully connected layers*, neither compilation nor the design space
  exploration supports fully connected layers.

## Dependencies

- protoc-3.0.0 compiler (libproto-java.jar is not required)
- libprotobuf.so (install this in your `LD_LIBRARY_PATH`)
- caffe (for generating test cases)
- maxcompiler 2015-2

## Project Structure

```
descriptors/  # Several example neural networks
protos/   # protobuf specification used throughout the project.
  fpgaconvnet/
    protos/
      parameters.proto  # Protobuf specification for neural networks
projects/  # Where several generated projects are located
scripts/   # Helper scripts for generating Makefiles etc.
src/
  fpgaconvnet/
    modelling/  # C++ Code used for design space exploration
    *.h         # Header files and source files for libraries
    *.cpp       #   used to facilitate communication between the 
                #   CPU and the FPGA, including uploading bistreams
                #   and writing to off-chip memory.
  java/   # Maxj code used for the design.
    kernels/
    lib/
    maxpower/
    *.maxj
  javatest/  # Various kernel tests (in .maxj format)
template/  # Template files used in generating projects
test_data/  # Data used for testing
```

## Usage

### Step -1: Setting up your project environment

1. Clone this repo: `git clone https://github.com/fyquah95/fpgaconvnetmaxeler`
2. Set the `FPGACONVNET_JAVA8_HOME` environment variable to a valid JDK 8 (or greater) installation path.
You can download java8 from oracle's website. _I am not entirely sure if this
is actually needed, but in my setup, `maxjc` required a java 1.7 (ecj.jar is a java 7 library),
whereas `maxJavaRun` required java 1.8 (because `MaxCompiler.jar` in 2018-1 is a java1.8 library).
I tried using a custom java installation (I placed java8 in `$HOME/jdk-...`) and configured the
PATH variable as appropriate. This doesn't work (`Unable to resolve type String` - implying
that java cannot find the JRE runtime libraries).  I suspect that this should work if the global
java installation is 1.8 (I have not tested this)._ You _must_ use Java 8 in oracle, as the
maxcompiler libraries makes use of `sun.*` libraries. Some of the libraries are deprecated in
Java 10, per se.
3. Refresh your shell environment eg: `source ~/.bashrc`
4. If you are targetting AWS F1 instances, you will need to configure your AWS credentials. See
   maxeler's guide for getting started on AWS EC2 instances for the relevant instructions.
5. Enter the project directory `cd fpgaconvnetmaxeler`

### Step 0: Write the CNN's Protobuf Desciptor

The protobuf descriptor is given by `protos/fpgaconvnet/protos/parameters.proto`.
You are required to define a `Network` protobuf. Remember to specify the number
of available fpga in `num_fpga_used`.

Refer to `descriptors/lenet.prototxt` for an example.

### Step 1: Generate a project

```bash
python ./scripts/create_project \
  --nets my_net.prototxt \
  --dir projects/my_net
  --max-ring  # This allows the usage of multiple fpgas.
  my_net
```

This creates a project in `projects/my_net` (which is, by default, tracked by
git). In the projects, you will see several files:

```bash
build/
  Makefile  # A generated Makefile. This Makefile, on its own doesn't contain
            # any targets. The targets are included via other Makefiles. You
            # can add custom targets, as long as you don't overwrite the original
            # contents.
src/
  main.cpp  # A default executable that is generated. This passes a random stream
            # of numbers into the generated network, and does not check the output.
            # You (probably) want to modify this, unless you are interested only
            # in performance numbers and is 100% sure about the model's correctness.
descriptors/
  my_net.prototxt  # The net you have created. This is a copy of the file you have
                   # made earlier, not a symlink / hard link.
```

### Step 2: Compiling

Depending on your requirements, you will want to modify `projects/my_net/src/main.cpp`
based on your needs.

Firstly, you need to perform DSE and generate the relevant targets.

```bash
make optimize  # Design space exploration generates <net>.optimized.prototxt
make gen_makefile  # This generates a Makefile, based on the number of FPGA required.
                   # it is possible to use less than the available FPGA to have
                   # better throughput.
```

### Step 4: Simulation

(Optional, but highly recommended) Secondly, you want to simulate the maxfile.

```bash
make sim
make runsim
```

### Step 4: DFE

Once you are ready, you can compile the net into the DFE (This will take awhile)
and you can run it. The default `main.cpp` file will run inference twice and report
their execution times.

```bash
make dfe
make rundfe
```

In the unlikely scenario where your compilation fails (either at MPPR or mapping),
you ought to relax the compilation resource usage threshold. This can be done by
by specifying the percentage of resource usage that ought to be considered at
design space exploration. An example of such specification can be found [here](projects/vgg_16_single/descriptors/vgg_16.prototxt)

## Examples

- lenet [multi FPGA](projects/lenet_maxring), [with reconfiguration](projects/lenet_single)
- cifar10 [multi FPGA](projects/cifar10_maxring), [with reconfiguration](projects/cifar10_single)
- alexnet [multi FPGA](projects/alexnet_maxring), [with reconfiguration](projects/alexnet_single)
- vgg16 [multi FPGA](projects/vgg_16_maxring), [with reconfiguration (Not working yet)](projects/vgg_16_single/)

## License

TBC
