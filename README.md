# comfyui-anima-snapref-fork

ComfyUI node for **SnapRef**, a reference adapter for Anima: give it a picture of a character and prompt any pose
or scene.

This is the node from [ArcAnima/SnapRef](https://github.com/ArcAnima/SnapRef) (`comfyui_nodes/snapref/`) with small
compatibility fixes so it works together with nodes that wrap Anima's attention, such as
[comfyui_anima_artist_cross_attn_mix](https://github.com/Localsmile/comfyui_anima_artist_cross_attn_mix).
SnapRef itself is the work of its original author; see the upstream repo for the model card, training code and
examples. The weights are **not** included.

## Installation

1. Install **Anima base v1.0** with its text encoder and VAE.
2. Clone this repo into `ComfyUI/custom_nodes/` (remove any older SnapRef node first):
   ```bash
   cd ComfyUI/custom_nodes
   git clone https://github.com/Localsmile/comfyui-anima-snapref-fork
   ```
3. Put the weights in `ComfyUI/models/loras/` (each also comes as an FP8 file):

   | weights | |
   |---|---|
   | [`snapref_v1a.safetensors`](https://huggingface.co/0ppxnhximxr/snapref-v1a) | recommended |
   | [`snapref_v2a_pre2.safetensors`](https://huggingface.co/0ppxnhximxr/snapref-v2a-pre2) | preview, stronger `mode char` |
   | [`snapref_v2a_pre1.safetensors`](https://huggingface.co/0ppxnhximxr/snapref-v2a-pre1) | preview, earlier checkpoint |
   | [`snapref_v1b.safetensors`](https://huggingface.co/0ppxnhximxr/snapref-v1b) | twice the LoRA rank of v1a |

4. Restart ComfyUI. Example workflows are on the Hugging Face pages under `workflows/`.

SnapRef is not a regular LoRA: load it with a SnapRef node, not with LoraLoader.

## Nodes

| node | inputs |
|---|---|
| **SnapRef** | one or two pictures picked inside the node, Mode (*Character & Style* / *Character*), Strength per picture, Fidelity |
| **SnapRef (Advanced)** | every setting: `mode`, `ref_in_uncond`, `strength`, `ref_caption`, `ref_guidance`, `image_2`/`image_3`, `weight_1`–`3`, `fidelity`, ... |

Both go between the model loader and the sampler. Write only the scene in the prompt. See the
[upstream README](https://github.com/ArcAnima/SnapRef#quick-start) for every setting.

## Changes from upstream

- **Works with nodes that wrap the attention modules.** cross_attn_mix swaps `block.cross_attn` for a wrapper that
  keeps the original as `.orig` during each model call. Upstream then fails with
  `AttributeError: _MixedCrossAttn has no attribute 'output_proj'`. SnapRef now looks through `.orig` when it attaches
  its LoRA hooks.
- **The reference stream always uses the stock cross-attention**, so it reads its own `ref_caption` and not another
  node's cached text K/V.
- **Per-branch `cond_or_uncond`.** When SnapRef runs the cond and uncond halves separately, each half gets its own
  `cond_or_uncond`, so per-branch code downstream (an artist mix that applies only to cond) sees the right branch.

Without such nodes the behaviour is the same as upstream.

## Troubleshooting

**Black images.** The output is NaN. Anima is a bf16 model and overflows in fp16. `--fast fp16_accumulation` makes
ComfyUI load the model in fp16; remove that flag (or add `--bf16-unet`) and check the log says
`model weight dtype torch.bfloat16`.

**Smeared images with the native text encoder.** Negative weights such as `(tag:-2)` are NegPiP syntax. The native
`CLIPTextEncode` multiplies the embedding by the weight as is, which breaks the conditioning. Move those tags to the
negative prompt when using the native encoder.

**Artist styles get lost.** With *Character & Style* (`cs`) the reference also sets the style. Use *Character*
(`char`) when the style should come from artist tags or an artist-mix node.

## License

- **Code:** Apache-2.0 ([LICENSE](LICENSE)), as upstream. `__init__.py` is modified as listed above.
- **Weights:** a derivative of the CircleStone Model, under the CircleStone Labs Non-Commercial License v1.2 (see the
  [model cards](https://huggingface.co/0ppxnhximxr/snapref-v1a)). Not affiliated with or endorsed by CircleStone Labs.
