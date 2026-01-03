# Flux + ControlNet + Qwen-VL Prompt Gen + Styles + Upscale (v1)

**Workflow JSON:** `workflows/flux_controlnet_qwen-vl_styles_upscale_v1.json`  
**Author/Curator:** Lairissa Lee® / RockyMtnBabe®  
**Goal:** Start from an input image → generate a descriptive prompt with Qwen-VL (optional) → optionally add a style → run a 2-stage Flux workflow with ControlNet hint prep (Canny/Depth) → decode + upscale.

---

## What this workflow does (high level)

1. **Load Image** (Node **17**)  
2. **Optional: Qwen-VL generates a detailed description** from the image (Node **163**)  
3. **Optional: Style selection + prompt assembly** (Nodes **153**, **143**, **136**, **144**, **139**)  
4. **ControlNet hint image prep** (resize + generate Canny/Depth hints) (Nodes **122**, **120**, **121**, **18**, **124**, **125**)  
5. **2-stage sampling**  
   - Stage A sampler (Node **53**) → quick preview decode (Nodes **55**, **49**)  
   - Stage B sampler (Node **97**) → final decode + save (Nodes **99**, **102**)  
6. **Upscale** + save + A/B compare (Nodes **157**, **156**, **159**, **162**, **160**)

---

## Main toggles / “how to drive it”

### A) Use Qwen prompt generation vs manual prompt
- **Qwen image → text:** Node **163** (“AILab_QwenVL”)  
  - Output goes to Node **164** (ShowText) and into the prompt switcher.
- **Manual prompt fallback:** Node **141** (“easy positive” – Alternative Stand-alone Prompt)

**Prompt switch:** Node **136** (“Any Switch (rgthree)”)  
- Input `any_01` is the Qwen description (from Node **163**)  
- Input `any_02` is your manual prompt (from Node **141**)  
Pick whichever is “active” by wiring/using the node’s selection behavior.

> Tip: If Qwen is too verbose, keep it on, but reduce it with a short “Prompt Preamble” (Node **143**) + style blocks (Node **153**) rather than trying to manually edit the huge description every time.

---

### B) Style selection (optional)
- **Node 153:** “Prompt Multiple Styles Selector” (WAS node suite)  
  Outputs “positive_string” that gets merged into the final prompt via Node **144** (Text Concatenate).

If you want this to work on other machines:
- You likely need to point WAS’s `webui_styles` to your repo `styles.csv` (or whatever you name it).
- Put `styles.csv` at repo root (recommended) and explain how to set the path in WAS config.

---

### C) Canny vs Depth hint switching
- **Node 125:** “Canny / Depth - SWITCH” (Crystools)
  - `false` currently selects the “on_false” input (Depth)  
  - `true` selects Canny

Upstream:
- **Node 18:** Canny (thresholds are in the node)
- **Node 124:** MiDaS DepthMap preprocessor (resolution input is fed from Node **120**)

The hint image feeding both preprocessors is normalized/resized by:
- **Node 121:** “HintImageEnchance” (collapsed; uses SDXL rec resolution inputs)

---

## ControlNet “strength” in *this* workflow

Important nuance: in this JSON, the “ControlNet Slider” is implemented as **sampler step control**, not a typical “strength” parameter:

- **Node 58:** “Maximum Steps” (Primitive INT) → feeds sampler steps (Nodes **53** and **97**)
- **Node 56:** “ControlNet Slider” (Primitive INT) → feeds:
  - Node **53** `end_at_step`
  - Node **97** `start_at_step`

So the slider is effectively:  
✅ “How many early steps should be influenced by the ControlNet stage” (Stage A), and when Stage B begins.

**Node 115** includes the rule of thumb: max slider value = max steps.

---

## The 2-stage sampling design (why there are two samplers)

- **Stage A (Node 53)** produces an intermediate latent and a quick decoded preview (Node 55 → Preview Node 49).
- That latent is then passed into **Stage B (Node 97)** for the remainder of the steps, using a different model loader (Node 31) and its own start step.

This is a common pattern for “structure first, detail later.”

---

## LoRAs in this workflow

- **Node 131:** “Power Lora Loader (rgthree)”
  - Currently includes toggles for:
    - `flux_realism_lora.safetensors`
    - `fluxRealSkin-V2.safetensors`
  - Both are OFF by default in the saved JSON.

Recommendation:
- Keep them OFF by default so the workflow runs “clean” on fresh installs.
- Put “suggested LoRAs” in `notes/` and don’t require them.

---

## Upscaling

- **Node 157:** Upscale model loader (`4x_NMKD-Siax_200k.pth`)
- **Node 156:** ImageUpscaleWithModel
- **Node 159:** ImageScaleBy (set to 0.5 right now — effectively downscales after upscaling)
- **Node 160:** Image Comparer (A/B)
- **Node 162:** SaveImage (to `ComfyUI-Upscale`)

> If you want “upscale and keep full size”, set Node 159 scale to `1.0` (or remove it). Right now it’s halving the result after the upscale.

---

## Required custom nodes (from the workflow metadata)

This workflow references:
- **rgthree-comfy** (labels, comparer, power lora loader, fast group muter, any switch)
- **ComfyUI-QwenVL** (AILab_QwenVL)
- **WAS Node Suite (Revised)** (Prompt Multiple Styles Selector, Text Concatenate)
- **ComfyUI-Easy-Use** (easy positive, easy showAnything)
- **ComfyUI-Crystools** (Switch any)
- **comfyui_controlnet_aux** (MiDaS, HintImageEnchance)
- **JPS Nodes** (SDXL recommended resolution calc)
- **KJNodes** (GetImageSizeAndCount)
- **comfyui-custom-scripts** (ShowText|pysssss)

If someone opens the workflow and sees “missing nodes”, it’s almost always one of the above.

---

## Models used in the JSON (what to install)

### UNET / diffusion models
- Node **31** loads: `flux1-dev-fp8.safetensors`
- Node **39** loads: `flux1DepthDevFp8_v10.safetensors` OR `flux1CannyDevFp8_v10.safetensors`

### Text encoders
- Node **34** DualCLIPLoader:
  - `clip_l.safetensors`
  - `t5xxl_fp8_e4m3fn.safetensors`

### VAE
- Node **32** loads: `ae.safetensors`

### Upscaler
- Node **157** loads: `4x_NMKD-Siax_200k.pth`

---

## Gotchas / repo hygiene

### 1) “Last used images” will show in the workflow UI
ComfyUI saves the last selected filename in the JSON (e.g., Node 17).  
That does **not** include your image file itself — unless you also commit the image into your repo.

**Best practice:**
- If you include example images, put them in `assets/` and keep them generic.
- Don’t commit anything private/personal.

### 2) styles.csv portability
If your style selector relies on a local path in `was_suite_config.json`, other users won’t have that path.  
Include a short “How to point WAS to styles.csv” section (above) and recommend they edit the config.

### 3) Qwen-VL model availability
Your Qwen node is set to a specific model option (`Qwen3-VL-2B-Instruct`). If a user doesn’t have it, they’ll need to download/configure the QwenVL node’s model setup.

---

## Recommended folder additions for this workflow

Inside `assets/`:
- `assets/flux_controlnet_qwen-vl_styles_upscale_v1/`
  - `preview_input.jpg` (optional)
  - `preview_output.jpg` (optional)
  - `workflow_screenshot.png` (optional)

Inside `notes/`:
- This doc (the one you’re reading)
- Optional: `INSTALL.md` with “required custom nodes + model links”

---

## Quick “how to run”

1. Load workflow JSON in ComfyUI.
2. Set Node **17** to your input image.
3. Decide prompt source:
   - Use **Qwen (Node 163)** or
   - Type a manual prompt in **Node 141**
4. (Optional) Pick a style in **Node 153**.
5. Choose hint type in **Node 125**: Canny vs Depth.
6. Set steps (**Node 58**) and controlnet slider (**Node 56**).
7. Queue prompt. Final output is saved by Node **102** (+ upscale output by Node **162** if enabled).
