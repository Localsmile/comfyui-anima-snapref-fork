# comfyui-snapref

ComfyUI node for **SnapRef**, a one-image reference adapter for Anima: give it a single picture of a character and
prompt any pose or scene.

This repository carries the SnapRef ComfyUI node from
[0ppxnhximxr/snapref-v1a](https://huggingface.co/0ppxnhximxr/snapref-v1a) with small compatibility fixes so it can be
used together with other Anima nodes. The adapter weights are **not** included; download them from Hugging Face.

> SnapRef is the work of its original author. This repo only adds the compatibility changes listed below.
> The idea builds on ElectronGirl's ComfyUI-Anima-Ref-Adapter (Apache-2.0).

## How it works (short)

The reference image runs once through its own branch (sigma 0, its own LoRA), and every block's self-attention
K/V are cached. During sampling each block's self-attention also attends to that cache, with a second LoRA on the
target branch. The reference sits at a separate RoPE position, so it is read for appearance, not copied as a
layout. Cost is about +15 % generation time.

## Installation

1. Install **Anima base v1.0** with its text encoder and VAE.
2. Clone this repo into `ComfyUI/custom_nodes/`:
   ```bash
   cd ComfyUI/custom_nodes
   git clone https://github.com/<your-name>/comfyui-snapref
   ```
3. Download an adapter into `ComfyUI/models/loras/` (subfolders are fine):
   - [`snapref_v1a.safetensors`](https://huggingface.co/0ppxnhximxr/snapref-v1a) (bf16, 382 MB)
   - `snapref_v1a_fp8.safetensors` (FP8, 195 MB, practically the same images)
   - or [SnapRef v1b](https://huggingface.co/0ppxnhximxr) (twice the LoRA rank)
4. Restart ComfyUI. Example workflows are in the Hugging Face repo under `workflows/`.

The adapter is not a regular LoRA: load it with the **SnapRef** node, not with LoraLoader.

## Usage

`SnapRef (MODEL, CLIP, VAE, IMAGE, adapter) → MODEL` — put it between the model loader and the sampler.

Write only the scene in the prompt; hair, eyes, outfit (and in `cs` mode the style) come from the reference.

| input | default | what it does |
|---|---|---|
| `mode` | `cs` | `cs` copies character and style. `char` copies only the character, so the style can come from artist tags |
| `ref_in_uncond` | off | off: only the positive branch reads the reference, so CFG pushes toward it. on: weaker, closer to the prompt |
| `ref_megapixels` | 1.0 | the reference is shrunk (never enlarged) to this area; trained at ≤ 1.0 |
| `strength` | 0 | log-scale attention bias on the reference. 1–2 for more detail (trims, accessories); non-zero is ~1.3× slower |
| `ref_caption` | empty | optional appearance tags of the reference (no name or artist) |
| `ref_guidance_on` | off | separate reference guidance, one extra model call per step until `ref_end` |
| `ref_guidance` | 4.5 | strength of that push; 6–8 for a closer match |
| `correction` | `apg` | keeps colors in check when `ref_guidance` is on |
| `ref_end` | 1.0 | last step with the extra call, as a fraction of denoising; ~0.6 is faster |

Recommended sampling: `euler` or `er_sde`, `simple`, 28–30 steps, cfg 4.5.

**Two characters:** stitch two references side by side (core *Image Stitch*), use `2girls` instead of `solo`, and
say in a few words who is on the left and who is on the right.

## Changes in this repo

Compared to the node shipped on Hugging Face:

- **Works with nodes that wrap the attention modules.** Some nodes (e.g.
  [comfyui_anima_artist_cross_attn_mix](https://github.com/Localsmile/comfyui_anima_artist_cross_attn_mix)) swap `block.cross_attn` for a wrapper that keeps the
  original as `.orig` during each model call. The original node failed with
  `AttributeError: _MixedCrossAttn has no attribute 'output_proj'`. SnapRef now looks through `.orig` when it
  attaches its LoRA hooks.
- **The reference stream always uses the stock cross-attention**, so it reads its own `ref_caption` and not another
  node's cached text K/V.
- **Per-branch `cond_or_uncond`.** When SnapRef runs the cond and uncond halves separately, each half now gets its
  own `cond_or_uncond` flag, so downstream per-branch code (such as an artist mix that applies only to cond) sees
  the right branch.

Without such nodes the behaviour is unchanged.

Tested with: ComfyUI v0.43.0, built-in Anima LLLite (`ModelPatchLoader` + `Apply Anima LLLite`),
comfyui_anima_artist_cross_attn_mix.

## Troubleshooting

**Black images.** The output is NaN. Anima is a bf16 model; in fp16 the SnapRef path overflows. Launching ComfyUI with
`--fast fp16_accumulation` makes ComfyUI prefer fp16 for the model. Remove that flag (or add `--bf16-unet`) and check
the log says `model weight dtype torch.bfloat16`.

**Smeared / painterly images with the native text encoder.** Negative prompt weights such as `(tag:-2)` are NegPiP
syntax for nodes that support it. The native `CLIPTextEncode` multiplies the embedding by the weight as is, which
breaks the conditioning. Move those tags to the negative prompt when using the native encoder.

**Artist styles get lost.** In `cs` mode the reference also sets the style. Use `mode: char` when the style should
come from artist tags or an artist-mix node; lower `strength` (e.g. −0.5) if the reference style still leaks in.

## Known limitations (from the original model card)

- Very small details (earrings, fine patterns, scars) can be simplified.
- A dark-skinned reference can come out with lighter skin.
- Some outdoor full-body prompts turn the bottom into a plain white floor; name the ground (grass, path).
- A scene can override the reference outfit (beach → swimsuit); raise `strength` or name the outfit.
- With two characters, attributes the references do not show clearly can mix between them.

## License

The SnapRef node and weights are distributed by their author under the
**CircleStone Labs Non-Commercial License v1.2**; the weights are a derivative of the CircleStone Model (Anima base
v1.0). See `LICENSE.md` and `NOTICE.txt` from the
[original repository](https://huggingface.co/0ppxnhximxr/snapref-v1a). **Non-commercial use only.** Not affiliated
with or endorsed by CircleStone Labs.

The post-CFG reference guidance (APG) follows ElectronGirl's ComfyUI-Anima-Ref-Adapter (Apache-2.0).
