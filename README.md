# FHE Convolution-Friendly Image Compression Pipeline

A Python/TenSEAL reimplementation and extension of the JPEG-inspired, CKKS-based image
compression scheme described in *"Convolution-Friendly Image Compression with FHE"*
(Mertens, Nicolas, Rovira — COSIC, KU Leuven / TII). Reference C++ implementation:
[KULeuven-COSIC/img-processing-fhe](https://github.com/KULeuven-COSIC/img-processing-fhe).

This repo contains a grayscale baseline plus a progression of RGB extensions that
reduce homomorphic bandwidth well below the naive "encrypt each channel separately"
approach, culminating in a version that supports 1024×1024 images.

## Idea in one paragraph

Images are split into `m×m` blocks (m = 8 or 16), leveled, DCT'd, quantized against a
JPEG-style quantization matrix, zigzag-scanned, and truncated to a **constant** number
of coefficients `c` per block (constant-time and content-independent, so the cutoff
itself never leaks structural information about the image). Each retained frequency
index is packed across every block into a single CKKS ciphertext via SIMD slots, so
total bandwidth is driven by `c` (the number of ciphertexts), not by image size.

## What's in this repo

| File | Description |
|---|---|
| `btp_local.py` | Grayscale baseline pipeline — compress, encrypt, homomorphic decompress → process (pixel-wise or 3×3 convolution) → recompress, decrypt, reconstruct. |
| `btp_local_rgb_maxpack_1024.py` | RGB extension: converts to YCrCb, subsamples chroma 4:2:0, and packs Y, Cr, and Cb into a **single** ciphertext set using pixel inversion as the pointwise op (the same affine transform for luma and chroma, so no per-channel masking is needed). Adds dynamic CKKS context sizing so 1024×1024 images are supported alongside 256/512. |
| `BTP_project_handoff.md` | Detailed design notes: parameter choices, the RGB packing progression, verified slot/chunk counts per image size and block size, and known caveats. |
| `images/` | Place your own test images here (see Usage below). Not included in this repo by default — see `.gitignore`. |

Earlier intermediate stages (naive 3-channel encryption, and a "Y separate / Cr+Cb
packed" 2c-ciphertext version) are described in the handoff document for anyone who
wants to reproduce the full comparison, even though only the final maximum-packing
version is kept as a runnable script here.

## Requirements

```bash
pip install tenseal numpy opencv-python scipy matplotlib scikit-image
```

Tested with Python 3.10+. `tenseal` provides the CKKS homomorphic encryption backend;
everything else is standard scientific Python.

## Usage

1. Create an `images/` folder next to the script you're running.
2. Add a **PNG** test image (JPEG sources introduce their own DCT compression and
   4:2:0 chroma subsampling, which compounds with this pipeline's own steps — see the
   in-code notes for details):
   - Grayscale pipeline: `images/si_{size}_gray.png` (e.g. `si_256_gray.png`)
   - RGB pipeline: `images/si_{size}_rgb.png` (e.g. `si_512_rgb.png`)
3. Run the script and answer the interactive prompts:

```bash
python btp_local_rgb_maxpack_1024.py
```

You'll be asked for:
- **Image size** — 256, 512, or 1024
- **Operation** — `Pixel` (pointwise inversion) or `Convolution` (3×3 sharpen)
- **Block size `m`** — 8 or 16
- **Quality `q`** — 1–99, controls quantization aggressiveness (50 = identity scaling)

The script prints per-stage timing, SSI (structural similarity) against the original,
compression ratio, and bandwidth in both directions, then displays the original vs.
reconstructed image side by side.

## Known limitations

- 1024×1024 with `m=8` needs TenSEAL's automatic multi-ciphertext chunking and does
  not reach the full 3× bandwidth saving that every other supported combination gets
  (still a real 25–33% saving vs. naive per-channel encryption). Use `m=16` at 1024px
  if the full 3× saving matters more than the smaller block size.
- 2048×2048 is not currently supported.
- See `BTP_project_handoff.md` for the full list of design trade-offs and caveats.

## License

Released under the MIT License — see [LICENSE](LICENSE).

## Acknowledgements

Based on the compression scheme and reference implementation from Mertens, Nicolas,
and Rovira's FHE image-processing paper and codebase (COSIC, KU Leuven / TII).
