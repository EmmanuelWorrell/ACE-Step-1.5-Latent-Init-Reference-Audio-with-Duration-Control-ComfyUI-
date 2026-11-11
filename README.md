# ACE-Step 1.5 — Latent-Init Reference Audio with Duration Control (ComfyUI)

A ComfyUI workflow for [ACE-Step 1.5](https://github.com/ace-step/ACE-Step) that generates a new song **starting from a reference track's audio latent** instead of from pure noise. The result keeps the reference's vocal timbre and instrumentation far more strongly than the standard "reference audio" conditioning does, while lyrics and tags still come from an LLM (Ollama) or from your own text.

This build also adds **duration control**: output length follows a "Length in seconds" value instead of being locked to the length of the reference file.

**Workflow file:** `Manny_ACE_DURATION_CONTROL.json`

---

## How it works

### Standard approach (what this replaces)

```
random noise  ──►  sampler  ◄── reference audio (soft conditioning)
```

The model starts from noise and is only *nudged* toward the reference. Timbre carries over weakly.

### Latent-Init ("hard" conditioning)

```
reference audio ─► VAEEncodeAudio ─► latent ─► sampler (denoise < 1.0) ─► VAEDecodeAudioTiled
```

The reference is encoded into the same latent space the sampler works in, and the sampler starts from that latent. `denoise` sets how much of the reference survives:

| denoise | Effect |
|---|---|
| low (~0.3–0.5) | Stays very close to the reference (closer to a remix/cover) |
| mid (~0.5–0.7) | Reference timbre and feel remain; the model has room to rewrite structure |
| 1.0 | Reference latent is effectively ignored (same as starting from noise) |

### Duration control

`VAEEncodeAudio` encodes whatever audio it receives, so the latent length equals the file length. To decouple them:

```
LoadAudio ─► AudioConcat (A + A, loop once) ─► TrimAudioDuration ─► VAEEncodeAudio ─► samplers
                                                      ▲
                                  "Length in seconds" (same value that feeds
                                  TextEncodeAceStepAudio1.5 → duration)
```

- The reference is looped once so the requested length can exceed the file length.
- `TrimAudioDuration` cuts it to the requested length.
- The text encoder's `duration` and the latent length now come from the **same** value, so they always agree.

**Limit:** maximum output length is **2× the reference file**. Beyond that, chain another `AudioConcat`. Past the original file length, the loop point is a hard splice; the sampler usually smooths it, but you may hear it.

---

## Workflow layout

- **4 sampler slots**, each packaged as a subgraph (KSampler + `VAEDecodeAudioTiled`):
  1. Turbo
  2. SFT (or Turbo Shift1)
  3. Turbo XL
  4. SFT XL
- **Reference Audio → Latent-Init group:** `LoadAudio` → `AudioConcat` → `TrimAudioDuration` → `VAEEncodeAudio` → `latent_image` on all four samplers.
- **Turbo XL** additionally uses `ACEStep15XLReferenceAudio` for its positive conditioning.
- **Ollama section:** generates tags, lyrics and a song title from a prompt via a local Ollama server.
- **Outputs:** `SaveAudioMP3`, `SaveAudioWithTags`, `PreviewAudio`, plus optional lyrics/data `.txt` export.
- **Unused nodes:** `EmptyAceStep1.5LatentAudio` (old noise-start latent) and `AudioSeparateVocals` are left in the graph but not connected to any output. Unconnected nodes are not executed.

---

## Requirements

**ComfyUI:** a recent version with ACE-Step 1.5 support and the built-in audio nodes (`TrimAudioDuration`, `AudioConcat`, `VAEEncodeAudio`).

**Custom node packs** (install via ComfyUI-Manager):

| Pack | Used for |
|---|---|
| [comfyui-ollama](https://github.com/stavsap/comfyui-ollama) | Tag / lyric / title generation |
| [rgthree-comfy](https://github.com/rgthree/rgthree-comfy) | Power Puter, Any Switch, Power Lora Loader, bypassers |
| [ComfyUI-Easy-Use](https://github.com/yolain/ComfyUI-Easy-Use) | `showAnything`, `saveText`, cache/GPU cleanup |
| comfyui-audio-expo | `SaveAudioWithTags` |

`ACEStep15XLReferenceAudio` and `AudioSeparateVocals` carry no pack metadata in the file. If ComfyUI reports them missing, use **Manager → Install Missing Custom Nodes**.

**Models** (place in the usual ComfyUI folders; the filenames below are what the workflow currently selects, so change them in the loader nodes to match yours):

- VAE: `ace_1.5_vae.safetensors`
- Text encoders (DualCLIPLoader): `qwen_0.6b_ace15.safetensors`, `qwen_4b_ace15.safetensors`
- Diffusion models, one per slot (Turbo, Turbo Shift1, XL Turbo merge, XL base/turbo merge). Use whichever ACE-Step 1.5 checkpoints you have; you do not need all four.

**Ollama:** a running server (default `http://127.0.0.1:11434`) with the model named in the connectivity node pulled. Not needed if you supply tags and lyrics manually.

---

## Usage

1. Load `Manny_ACE_DURATION_CONTROL.json` in ComfyUI.
2. In **Load Reference Audio**, select your reference track (WAV or MP3).
3. Set the **Length in seconds** value (node "Duration in Seconds"). This drives both the text encoder and the latent length.
4. Provide tags/lyrics (via Ollama or manually).
5. Pick which sampler slot(s) to run (use the group bypassers) and set `denoise` per sampler.
6. Queue the prompt.

### Tuning notes

These come from the author's testing, not a guarantee. Results depend heavily on the track.

- **Denoise is per sampler.** Turbo samplers run few steps, so a given denoise leaves very few effective steps; they behave differently from SFT. Tune each branch separately and change one setting at a time.
- **Too close to the reference?** Raise `denoise`.
- **Losing the voice / gibberish vocals?** Lower `denoise`.
- **Turbo / distilled models and CFG:** very high CFG (e.g. 10) tends to degrade coherence on Turbo. Keep Turbo CFG modest and use higher CFG on SFT.
- **Genre not following the prompt:** raise CFG on the SFT branches.

---

## Known issues

- **Lyric adherence.** With latent-init plus `ACEStep15XLReferenceAudio`, lyrics can be skipped or sung out of order, sometimes worse than standard ACE-Step. The reference latent carries its own timing, which competes with the new lyrics. Things to try, one at a time:
  - Make sure "Length in seconds" matches the length actually being generated (this build does this by design).
  - Lower Turbo CFG.
  - Lower `reference_strength` on `ACEStep15XLReferenceAudio`.
  - Point a sampler's positive input back at the plain `TextEncodeAceStepAudio1.5` output to see whether the reference conditioning is the cause.
- **Silent output for some files.** Some WAV/MP3 files load as silence. Re-encode first: `ffmpeg -i input.mp3 -ar 44100 -ac 2 -c:a pcm_s16le output.wav`.
- **Output ≤ 2× reference length** (see above).
- **Voice cloning is not guaranteed.** This is style/timbre transfer through a diffusion model, not a dedicated voice-conversion system.

---

## Credits

- [ACE-Step](https://github.com/ace-step/ACE-Step) for the base model.
- ComfyUI, rgthree, ComfyUI-Easy-Use and comfyui-ollama authors for the node packs used here.
- Workflow author: Emmanuel Worrell
- Original Workflow author: Tremolo

## License

_Add a license for the workflow file (e.g. MIT). Model weights and custom node packs have their own licenses; check them before commercial use._

## Disclaimer

Only use reference audio you have the right to use. Generated output that imitates a real artist's voice may raise legal and ethical issues depending on where and how it is used.
