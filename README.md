# A set of python helpers to convert [TF variant of openai/whisper](https://huggingface.co/openai/whisper-base) ASR ML model to TF lite

## Purpose

The main goal this distribution pursues is to build and test the conversion of the
[openai/whisper](https://huggingface.co/openai/whisper-base) model to the tflite format.

Initially this goal seems to be quite straightforward, but in fact it is not. The widely known
approach to wrap the huggingface model with the class with the tf.function decorated serving method
and then use the `tf.lite.TFLiteConverter` to convert the model fails on the tiny models which are
most valuable in the TF lite targeted environments.

Besides this task the helper functions suggest the LogMel Spectrogram extraction and solving the notorious
crash problem with the weird message like this one:

```
RuntimeError: tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::apply(optimized_ops::Mean<T, U>, args) was not true.tensorflow/lite/kernels/reduce.cc:445 std::appl...
```
## Usage

```shell
pip install git+https://github.com/s4ysolutions/whisper-s4y
```

There're colab notebooks:

with the whole  pipeline <a target="_blank" href="https://colab.research.google.com/github/s4ysolutions/whisper-s4y/blob/main/colab/whisper_tflite_s4y.ipynb">
  <img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>
</a>

and the sample to create TFLite model for the featres extractor <a target="_blank" href="https://colab.research.google.com/github/s4ysolutions/whisper-s4y/blob/main/colab/whisper_s4y_usage.ipynb">
  <img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>
</a>



## Overview

The whole model is treated as pipeline with the following stages:

1. **Features extraction** - the audio signal is converted to the Mel spectrogram
2. **Encoding** - the Mel spectrogram is passed to the encoder model
3. **Decoding** - the encoder output is passed to the decoder model which produces the list of the tokens
4. **Postprocessing** - the list of the tokens is converted to the text

The original Huggingface model encapsulates the encoder and decoder and applying the same optimization to the both
models causes either the crash or the unacceptable quality of the output.

The idea is to split the model into the encoder and decoder and customize the conversion process for each of them.

Next the tflite models run separately and the output of the encoder is passed to the decoder.

## Detailed

### Audio signal

The audio signal is expeted to be mono 16K waveform represented as the tf.Tensor(1,N)

```python
import tensorflow_io as tfio
import tensorflow as tf

waveform, sample_rate = tf.audio.decode_wav(tf.io.read_file(wav_file_path))
waveform = tfio.audio.resample(waveform, rate_in=sample_rate, rate_out=16000)
```

see [normalized_audio_from_wav.py](https://github.com/s4ysolutions/whisper-s4y/blob/e4bef88943c00e7c2b111738c1c79caa809d16b7/tests/__init__.py#L86)

### Features extraction

```python
from whisper_s4y.features_extractor import S4yFeaturesExtractor

extractor = S4yFeaturesExtractor()
input_features = extractor(waveform)
```

see [test_features_extractor.py](./tests/units/test_features_extractor.py)

see [test_features_extractor_tflite.py](tests/units/test_features_extractor_tflite.py)

alternatively the features can be extracted with the wrapper around the `huggingface` API:

```python
import whisper_s4y.whisper.huggingface as hf

feature_extractor = hf.feature_extractor(huggingface_model_id)
input_features = feature_extractor(waveform, sampling_rate=16000, return_tensors="tf")["input_features"]
```

**Note:** S4yFeaturesExtractor should be treated just as a PoC, and the idea and should not be used in the
production code because in order to be convertable to TF Lite it is implemented with `tf.signal.stft` which does not
allow to have `FFT_LENGTH = 400` as the whisper model requires. S4yFeaturesExtractor set `FFT_LENGTH = 512`
(the closest power of 2) which causes the quality degradation.

### Encoder

```python
from whisper_s4y.whisper.huggingface.s4y_model import S4yEncoder

encoder = S4yEncoder(huggingface_model_id)
last_hidden_states = encoder(input_features)
```

see [test_encoder.py](./tests/units/test_encoder.py)

see [test_encoder_tflite.py](tests/units/test_encoder_tflite.py)

### Decoder

```python
from whisper_s4y.whisper.huggingface.s4y_model import S4yDecoder

decoder = S4yDecoder(test_model_id, lang='ar', max_length=100)
tokens = decoder(encoder_hidden_states=last_hidden_states)
```

see [test_decoder.py](./tests/units/test_decoder.py)

see [test_decoder_tflite.py](tests/units/test_decoder_tflite.py)

### Postprocessing

Translating the tokens to the text can not be implemented as TF graph and another huggingface wrapper should be used.

```python
from whisper_s4y.whisper import huggingface as hf

tokenizer = hf.tokenizer(test_model_id)
print(tokenizer.decode(tokens))
```

### Combined models

Besides the above models there are few combined ones:

- `S4yPcmEncoder` - the model which converts the PCM signal to the encoded output ready to be decoded
- `S4yEncoderDecoder` - the encoder and decoder are combined in the single model ready to be fed with the LogMel
  spectrogram
- `S4yGenerator` - the model which uses traditional wrapper around the huggingface model and should be used to convert
  the non-tiny whisper models to the TF Lite format 

```python

## Converting to TF Lite

Each of the models have `tflite()` method which converts the model to the TF Lite format and saves it to the file.

```python
from whisper_s4y.whisper.huggingface.s4y_model import S4yEncoderDecoder

tflite_model_path = S4yEncoderDecoder(test_model_id, lang=lang).tflite(log=test_log, optimize=optimize)
interpreter = tf.lite.Interpreter(model_path=tflite_model_path)

runner = interpreter.get_signature_runner()
tokens = runner(input_features=transformer_input_features)['tokens']
```

## Crash problems

The converted models often crash with the weird messages. The one reason is using the `-inf` values in the
implementation of Hugginfface transformers. At least the one place is detected and patched in [`src/whisper_s4y/whisper/huggingface/__init__.py`](https://github.com/s4ysolutions/whisper-s4y/blob/d7b6952c80ac91664bf2ddc211ae3edeb2bd84a6/src/whisper_s4y/whisper/huggingface/__init__.py#L8)
with 

```python
from whisper_s4y.whisper.huggingface import tf_logits_process_fix as fix

TFForceTokensLogitsProcessor.__init__ = fix.TFForceTokensLogitsProcessor_patched__init__
TFForceTokensLogitsProcessor.__call__ = fix.TFForceTokensLogitsProcessor_patched__call__
```

... and some [more](https://github.com/s4ysolutions/whisper-s4y/commit/3894a5f406e0d6fd4616847708896122b1d1f08b), but the other places can exist.
