# Node Parameters Documentation for Skimmed_CFG

This document provides a detailed explanation of all the nodes and their parameters within the Skimmed_CFG repository. These nodes are designed for ComfyUI to allow for higher CFG scales in latent diffusion models by mitigating the "burning" effect.

## Common Parameter Types

- **MODEL**: This is the input model, typically from a model loader. All nodes in this repository take a model as input and output a modified model.
- **FLOAT**: A floating-point number.
- **BOOLEAN**: A true/false value.
- **STRING**: A text string.

## General Notes from README

- **Skimming Scale**: This is a crucial concept. Lower values (e.g., 2-3) provide maximum anti-burn effect. A moderate value (e.g., 4) is considered a cruise scale. Higher values (e.g., 5-7) can lead to more colorful or stronger styles but might reintroduce some burning.
- **SDE Samplers**: Can still burn a little, but much less with these nodes. They might produce nonsensical results with low steps, except for the "Timed flip" node.
- **Steps**: A too-low skimming scale might necessitate more steps.
- **Negative Prompts**: A good negative prompt often focuses on style.
- **Super High Scales**: It might be beneficial to cut the negative prompt before the end using a node like "Support empty uncond" and setting the `end_at` parameter in `ConditioningSetTimestepRange` to around 65% to avoid potential artifacts.

---

## 1. Skimmed CFG

**Class Name**: `CFG_skimming_single_scale_pre_cfg_node`

This is the primary node for applying the CFG skimming technique. It adjusts the conditional and unconditional guidance to prevent over-exposure or "burning" at high CFG scales.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model to be patched.
    -   **Practical Use**: Connect this to the output of a model loader or another model-modifying node.

-   **`Skimming_CFG`**:
    -   **Type**: `FLOAT`
    -   **Default**: `7`
    -   **Min**: `0`
    -   **Max**: `10` (defined by `MAX_SCALE`)
    -   **Step**: `0.5` (defined by `1 / STEP_STEP`)
    -   **Round**: `0.01`
    -   **Tooltip**: "The fallback scale for the 'bad' values."
    -   **Practical Use**: This is the main control for the skimming effect. It determines the CFG scale applied to parts of the image that are identified as problematic or prone to burning.
        -   Lower values (e.g., 2-3 recommended for max anti-burn) will more aggressively reduce burning, potentially requiring more steps.
        -   A value of `4` is suggested as a "cruise scale."
        -   Higher values (e.g., 5-7) can result in more colorful and stronger styles but with less anti-burn effect.
        -   If set to a negative value, it will use the main `cond_scale` (from the sampler).

-   **`full_skim_negative`**:
    -   **Type**: `BOOLEAN`
    -   **Default**: `False`
    -   **Description**: When true, it more aggressively skims parts of the conflicting influence, especially in the negative conditioning. If `Skimming_CFG` is set to `0` and this is true, it will effectively set the "bad" values in the negative prediction to a CFG scale of 0.
    -   **Practical Use**: Can be used for a stronger anti-burn effect. The README mentions using this with `disable_flipping_filter` for specific effects (e.g., "razor skim" or the last example image with CFG 32).

-   **`disable_flipping_filter`**:
    -   **Type**: `BOOLEAN`
    -   **Default**: `False`
    -   **Description**: When true, it disables a filter that checks for sign changes in predictions, giving the `Skimming_CFG` more direct control.
    -   **Practical Use**: Meant to be used with `full_skim_negative` toggled on for more pronounced skimming. It changes how "outer influence" is determined in the `get_skimming_mask` function. When false (default), the filter considers deviation influence, matching prediction signs, and matching differences after scaling. When true, it only considers matching prediction signs and differences after scaling.

-   **Internal/Code-only Parameters (not directly exposed in UI but used by other nodes or internally)**:
    -   `start_at_percentage`: (Default: 0) Percentage of sampling steps at which to start applying the patch.
    -   `end_at_percentage`: (Default: 1) Percentage of sampling steps at which to stop applying the patch.
    -   `flip_at_percentage`: (Default: 0) Percentage of sampling steps at which to flip the `disable_flipping_filter` logic (used by `skimFlipPreCFGNode`).

---

## 2. Skimmed CFG - Timed flip

**Class Name**: `skimFlipPreCFGNode`

This node is described as less of an anti-burn and more of an enhancer. It's particularly effective with SDE Samplers and aims to enhance randomness and overall image quality by flipping the skimming filter behavior during the sampling process.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model.
    -   **Practical Use**: Connect to a model loader.

-   **`flip_at`**:
    -   **Type**: `FLOAT`
    -   **Default**: `0.3`
    -   **Min**: `0.0`
    -   **Max**: `1.0`
    -   **Step**: `0.05` (1/20)
    -   **Round**: `0.01`
    -   **Tooltip**: "Relative to the step progression. Completely at 0 will give smoother results. Completely at one will give noisier results. The influence is more important from 0% to 30%."
    -   **Practical Use**: Determines at what point in the sampling process the `disable_flipping_filter` logic (from the base `CFG_skimming_single_scale_pre_cfg_node`) is inverted.
        -   A value of `0` means the filter logic is flipped from the start (equivalent to `disable_flipping_filter = not reverse`).
        -   A value of `1` means the filter logic is never effectively flipped during the main sampling steps (it would flip only at the very end, having minimal impact).
        -   Values between `0` and `1` will flip the logic partway through. The tooltip suggests the 0% to 30% range (0.0 to 0.3) has the most significant influence.
        -   This node internally sets `full_skim_negative=True` and passes the `reverse` parameter as the initial state for `disable_flipping_filter`.

-   **`reverse`**:
    -   **Type**: `BOOLEAN`
    -   **Default**: `False`
    -   **Tooltip**: "If turned on you will obtain a composition closer to what you would normally get with no modification."
    -   **Practical Use**: Sets the initial state of the `disable_flipping_filter` before the `flip_at` percentage is reached. If `False` (default), `disable_flipping_filter` starts as `False` and then flips to `True` at `flip_at` percentage. If `True`, `disable_flipping_filter` starts as `True` and flips to `False`. This effectively changes which part of the sampling process uses the more aggressive skimming.

---

## 3. Skimmed CFG (Constant Skim)

**Class Name**: `constantSkimPreCFGNode`

This node appears to be a simplified wrapper for the main "Skimmed CFG" node, applying a fixed configuration. It always uses `Skimming_CFG=-1` (meaning it defaults to the sampler's CFG scale for fallback), `full_skim_negative=True`, and `disable_flipping_filter=False`.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model.
    -   **Practical Use**: Connect to a model loader.

-   **`enabled`**:
    -   **Type**: `BOOLEAN`
    -   **Default**: `True`
    -   **Practical Use**: A simple toggle to turn the effect of this node on or off. If `False`, it returns the original model untouched.

---

## 4. Skimmed CFG - replace

**Class Name**: `skimReplacePreCFGNode`

This node modifies the unconditional guidance (negative prompt) by replacing certain values with those from the conditional guidance (positive prompt). The goal is to nullify (or set to an equivalent CFG scale of 1) the effect of values targeted by the skimming filter.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model.
    -   **Practical Use**: Connect to a model loader. It directly modifies the `uncond` tensor based on `cond` where the `skim_mask` identifies conflicting areas. It applies this logic from both `cond` to `uncond` and `uncond` to `cond` perspectives.

**Effect on Image**:
The README shows an example with CFG 100. This method tends to produce images with strong adherence to the positive prompt in areas identified by the mask, potentially leading to sharp and defined features but might lose some of the negative prompt's influence in those specific areas.

---

## 5. Skimmed CFG - linear interpolation

**Class Name**: `SkimmedCFGLinInterpCFGPreCFGNode`

Instead of directly replacing values (like the "replace" node), this node performs a linear interpolation between the conditional and unconditional values in the areas targeted by the skimming filter. This is highlighted as "Highly recommended!" in the README.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model.
    -   **Practical Use**: Connect to a model loader.

-   **`Skimming_CFG`**:
    -   **Type**: `FLOAT`
    -   **Default**: `5.0`
    -   **Min**: `0.0`
    -   **Max**: `10` (defined by `MAX_SCALE`)
    -   **Step**: `0.5` (defined by `1 / STEP_STEP`)
    -   **Round**: `0.01`
    -   **Practical Use**: Controls the interpolation strength. The `fallback_weight` is calculated as `(Skimming_CFG - 1) / (cond_scale - 1)`.
        -   If `Skimming_CFG` is close to `1`, the `fallback_weight` is close to `0`, meaning the unconditional guidance in masked areas will be mostly replaced by the conditional guidance (`conds_out[0]`).
        -   If `Skimming_CFG` is close to `cond_scale` (the main CFG scale from the sampler), the `fallback_weight` is close to `1`, meaning the unconditional guidance in masked areas will be mostly preserved (`conds_out[1]`).
        -   Values in between will blend the conditional and unconditional guidance. The default of `5.0` provides a balance.

**Effect on Image**:
This method offers a softer transition than the "replace" node. It can help maintain some influence from the negative prompt while still mitigating burn and enhancing positive prompt adherence in targeted areas. The README example at CFG 100 shows a well-balanced image.

---

## 6. Skimmed CFG - linear interpolation dual scales

**Class Name**: `SkimmedCFGLinInterpDualScalesCFGPreCFGNode`

This node extends the linear interpolation concept by providing two separate skimming scales: one for when the mask is determined from `cond` vs `uncond` ("positive" application) and another for when the mask is determined from `uncond` vs `cond` ("negative" application). The names "positive" and "negative" are more for visual intuition than strict adherence to prediction types.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model.
    -   **Practical Use**: Connect to a model loader.

-   **`Skimming_CFG_positive`**:
    -   **Type**: `FLOAT`
    -   **Default**: `5.0`
    -   **Min**: `0.0`
    -   **Max**: `10` (defined by `MAX_SCALE`)
    -   **Step**: `0.5` (defined by `1 / STEP_STEP`)
    -   **Round**: `0.01`
    -   **Practical Use**: Controls the interpolation when the skimming mask is derived from `(x_orig, conds_out[0], conds_out[1], cond_scale)`. A higher value here tends to go towards high saturation. This scale influences how much `conds_out[1]` (unconditional) should be interpolated towards `conds_out[0]` (conditional) when `conds_out[0]` is considered the primary driver of the conflict.

-   **`Skimming_CFG_negative`**:
    -   **Type**: `FLOAT`
    -   **Default**: `5.0`
    -   **Min**: `0.0`
    -   **Max**: `10` (defined by `MAX_SCALE`)
    -   **Step**: `0.5` (defined by `1 / STEP_STEP`)
    -   **Round**: `0.01`
    -   **Practical Use**: Controls the interpolation when the skimming mask is derived from `(x_orig, conds_out[1], conds_out[0], cond_scale)`. A higher value for the *other* slider (positive) tends to go towards high saturations, and vice-versa with this slider. This scale influences how much `conds_out[1]` (unconditional) should be interpolated towards `conds_out[0]` (conditional) when `conds_out[1]` is considered the primary driver of the conflict (or, more accurately, when the roles of cond/uncond are swapped for mask generation).

**Effect on Image**:
Provides finer control over the interpolation process. The README examples (e.g., "Linear interpolation dual scale 10/0" vs "5/0") show how adjusting these can significantly impact saturation and contrast.
-   High `Skimming_CFG_positive` and low `Skimming_CFG_negative` might push colors and details from the positive prompt more strongly.
-   Low `Skimming_CFG_positive` and high `Skimming_CFG_negative` might result in a more muted or desaturated look where the negative prompt's characteristics are more preserved or where the positive prompt's influence is dampened in conflicting areas.

---

## 7. Skimmed CFG - Difference CFG

**Class Name**: `differenceCFGPreCFGNode`

This node implements an alternative algorithm based on the difference in output when using the main CFG scale versus a reference (typically lower) CFG scale. It aims to bring back details or characteristics that go "too far" or become exaggerated at high CFG scales by comparing them to a more stable, lower CFG scale output.

**Parameters**:

-   **`model`**:
    -   **Type**: `MODEL`
    -   **Description**: The input diffusion model.
    -   **Practical Use**: Connect to a model loader.

-   **`reference_CFG`**:
    -   **Type**: `FLOAT`
    -   **Default**: `5.0`
    -   **Min**: `0.0`
    -   **Max**: `10` (defined by `MAX_SCALE`)
    -   **Step**: `0.5` (defined by `1 / STEP_STEP`)
    -   **Round**: `0.01`
    -   **Practical Use**: This is the lower CFG scale used as a reference. The node calculates the difference between the image generated with the main `cond_scale` and this `reference_CFG`. The unconditional prediction (`conds_out[1]`) is then adjusted based on this difference, guided by the chosen `method`.

-   **`method`**:
    -   **Type**: `STRING` (Dropdown menu)
    -   **Options**: `["linear_distance", "squared_distance", "root_distance", "absolute_sum"]`
    -   **Default**: (The first option, likely "linear_distance", though not explicitly stated in code, it's common)
    -   **Practical Use**: Defines how the difference between the high-CFG and reference-CFG outputs is measured and applied.
        -   **`linear_distance`**: Calculates the absolute difference between denoised outputs at `cond_scale` and `reference_CFG`. This difference is normalized and used to interpolate the unconditional prediction.
        -   **`squared_distance`**: Similar to `linear_distance`, but the normalized difference is squared. This will emphasize larger differences more.
        -   **`root_distance`**: Similar to `linear_distance`, but the square root of the normalized difference is taken. This will give more weight to smaller differences.
        -   **`absolute_sum`**: Calculates a new effective scale based on the L1 norm (sum of absolute values) of the denoised outputs at `reference_CFG` and `cond_scale`. It then derives a `fallback_weight` to interpolate the unconditional prediction. This method tries to match the overall magnitude of change.

-   **`end_at_percentage`**:
    -   **Type**: `FLOAT`
    -   **Default**: `0.80`
    -   **Min**: `0.0`
    -   **Max**: `1.0`
    -   **Step**: `0.01`
    -   **Round**: `0.01`
    -   **Tooltip**: "Relative to the step progression. 0 means disabled, 1 means active until the end."
    -   **Practical Use**: Determines up to what point in the sampling process this difference CFG adjustment is applied. A value of `0.80` means it's active for the first 80% of the sampling steps. Setting it to `0` disables the node's effect. Setting it to `1` keeps it active throughout all steps. This can be useful to apply strong corrections early and let the regular sampling process refine details later.

**Effect on Image**:
This node can help in reducing artifacts and "overcooking" that can occur with very high CFG scales by reintroducing some characteristics of a lower, more stable CFG scale. The choice of method will affect how subtly or aggressively this correction is applied. For instance, `squared_distance` might make more dramatic changes where the high CFG deviates significantly from the reference.
