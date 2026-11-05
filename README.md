# Leaf classifier - live camera in the browser

**Open it on your phone:** <https://jerrykyr.github.io/mlr-ex08/>

A single ``index.html`` that classifies leaves straight from the phone's rear
camera using the exported ONNX model in this repository. Everything runs in the
browser as WebAssembly: the image is never sent anywhere. Works on Android and
iPhone; nothing to install.

[**https://jerrykyr.github.io/mlr-ex08/**](https://jerrykyr.github.io/mlr-ex08/)

## Model

| | |
|---|---|
| Architecture | MobileNetV3 (torchvision), exported with torch 2.11.0 |
| File | ``leaf_model.onnx`` (6.3 MB) |
| Classes | apple, cherry, chestnut, maple |
| Input | ``1 x 3 x 96 x 96`` float32, tensor name ``image`` |
| Output | tensor ``logits`` - raw logits, **no SoftMax node in the graph** |
| Normalisation | mean ``[0.485, 0.456, 0.406]``, std ``[0.229, 0.224, 0.225]`` |

``model_info.json`` is the authoritative description and is read by the page at
run time, so the preprocessing cannot drift from the model.

## Preprocessing actually performed, per frame

1. Take the **central square** of the frame (``side = min(width, height)``).
2. ``drawImage`` that square into a 96 x 96 canvas (crop + resize in one step).
3. ``getImageData`` -> RGBA bytes; keep R, G, B.
4. ``value = byte / 255`` -> 0..1.
5. ``value = (value - mean[ch]) / std[ch]`` for each of the three channels.
6. Store planar **CHW** (R plane, G plane, B plane) in a ``Float32Array`` of
   length 3 * 96 * 96, dims ``[1, 3, 96, 96]``.
7. **Softmax** over the four logits before displaying probabilities.

## Checking that it agrees with the notebook

The page has a built-in self-check (a button, or open the page as
``?selfcheck``). It loads ``check_image.jpg``, runs the identical pipeline and
prints its probabilities next to ``check_expected.json`` with the absolute
difference per class. Last measured: largest difference ``0.000000`` - the two
agree well beyond three decimals.

## Privacy

The camera image and any uploaded photo are decoded into a ``Float32Array`` and
handed to the WASM session in the same JavaScript context. Nothing is
serialised and nothing is sent out. The only requests the page makes are:

* ``https://cdn.jsdelivr.net/npm/onnxruntime-web@1.20.1/dist/`` - the ONNX
  Runtime WebAssembly engine (``ort.min.js`` plus its ``.wasm``/``.mjs``);
* the four same-origin files of this repository.

The page lists every URL it has requested and flags any request that carries a
body, so this can be checked rather than trusted.

The QR code at the bottom of the page is drawn by a QR encoder written into
``index.html`` itself (byte mode, ECC level M, versions 1-6, Reed-Solomon) -
no QR image service is used.

## Files

| File | Purpose |
|---|---|
| ``index.html`` | the whole app: camera, preprocessing, inference, QR code |
| ``leaf_model.onnx`` | the exported model |
| ``model_info.json`` | classes, input size, mean, std |
| ``check_image.jpg`` | 96 x 96 test image for the self-check |
| ``check_expected.json`` | expected probabilities for that image |
| ``REVIEW.md`` | answers to the four review questions |
| ``REPORT.md`` | report on task 4 |

## Notes

* ``.nojekyll`` is present so GitHub Pages serves ``leaf_model.onnx`` verbatim
  instead of running it through a Jekyll build.
* GitHub Pages does not send COOP/COEP headers, so ``SharedArrayBuffer`` is
  unavailable; the page pins ONNX Runtime to a single WASM thread, which is
  also the safe setting for iOS Safari.
* On iPhone the camera only starts from a tap, so the camera starts on the
  **Start camera** button; ``playsinline`` keeps the preview in the page.