# nam-binary-loader

A compact binary model format (`.namb`) for [Neural Amp Modeler](https://github.com/sdatkinson/NeuralAmpModelerCore) models.

## Motivation

NAM models are distributed as `.nam` files, which are JSON documents containing model configuration and weights encoded as arrays of floating-point numbers in text form. While JSON is convenient and human-readable, it has significant drawbacks in resource-constrained environments:

- **Parsing overhead**: The nlohmann/json library adds substantial code size and requires dynamic memory allocation during parsing. On embedded targets (e.g. STM32H7 with 512 KB RAM), this can be prohibitive.
- **File size**: Text-encoded floats are roughly 10x larger than their binary representation. A typical WaveNet model shrinks by ~80% when converted to `.namb`.
- **Load time**: Binary deserialization is essentially `memcpy` for weights, whereas JSON parsing must tokenize, validate, and convert each number from text.

The `.namb` format addresses these issues by storing model configuration as fixed-layout binary structures and weights as raw IEEE 754 floats. The loader (`get_dsp_namb`) has no dependency on nlohmann/json, making it suitable for embedded targets where only the binary loader needs to be linked.

## File format overview

All multi-byte values are little-endian. Format version: 1.

| Offset | Size | Field |
|--------|------|-------|
| 0 | 4 | Magic number (`0x4E414D42` = "NAMB") |
| 4 | 2 | Format version |
| 6 | 2 | Flags (reserved) |
| 8 | 4 | Total file size |
| 12 | 4 | Weights offset |
| 16 | 4 | Total weight count |
| 20 | 4 | Model block size |
| 24 | 4 | CRC32 checksum (IEEE 802.3, excludes this field) |
| 28 | 4 | Reserved |
| 32 | 48 | Metadata block (version, sample rate, loudness, levels) |
| 80 | var | Model block (architecture ID + config) |
| ... | ... | Padding (4-byte aligned) |
| weights_offset | weight_count * 4 | Weight data (raw float32) |

Supported architectures: Linear, ConvNet, LSTM, WaveNet (including recursive condition DSP).

## Repository structure

```
namb/                  # Library: format spec, reader/writer, loader
  namb_format.h        #   Binary format constants, CRC32, BinaryReader/BinaryWriter
  get_dsp_namb.h       #   Public API: get_dsp_namb()
  get_dsp_namb.cpp     #   Binary loader implementation
tools/
  nam2namb.cpp         # CLI converter: .nam (JSON) -> .namb (binary)
  loadmodel.cpp        # Model loader supporting both .nam and .namb
test/
  test_namb.cpp        # Tests: CRC32, format validation, round-trip, size reduction
```

## Dependencies

- [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore): at the moment, this project requires a modified version of NeuralAmpModelerCore that is available as this [pull request](https://github.com/sdatkinson/NeuralAmpModelerCore/pull/227)
- Eigen (transitive, via NeuralAmpModelerCore)
- nlohmann/json (only for `nam2namb` converter tool, **not** for the loader)

## Building

```bash
mkdir build && cd build
cmake .. -DNAM_CORE_PATH=/path/to/NeuralAmpModelerCore
make
```

`NAM_CORE_PATH` defaults to `../NeuralAmpModelerCore` (sibling directory).

## Usage

### Converting models

```bash
./nam2namb model.nam model.namb
```

If the output path is omitted, the `.nam` extension is replaced with `.namb`.

### Loading in C++

```cpp
#include <namb/get_dsp_namb.h>

// From file
auto model = nam::get_dsp_namb("model.namb");

// From memory buffer (useful for embedded/QSPI flash)
auto model = nam::get_dsp_namb(data_ptr, data_size);
```

### Running tests

```bash
./test_namb
```

Tests require example models from the NeuralAmpModelerCore `example_models/` directory to be accessible from the working directory.
