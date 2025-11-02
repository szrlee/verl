# Rollout Correction

**Author:** [Yingru Li](https://richardli.xyz/)

Last updated: 10/27/2025.

This document provides a comprehensive overview of the Rollout Correction implementation in verl.

**Note on Naming**: This feature is called "Rollout Correction" to reflect the complete functionality: importance sampling (IS) weights, rejection sampling (RS), and veto mechanism. The internal variable `rollout_is_weights` retains its name as it specifically refers to the IS weights component.

### BibTeX Citation

```bibtex
@misc{liu-li-2025,
  title = {When Speed Kills Stability: Demystifying RL Collapse from the Inference-Training Mismatch},
  url = {https://yingru.notion.site/When-Speed-Kills-Stability-Demystifying-RL-Collapse-from-the-Inference-Training-Mismatch-271211a558b7808d8b12d403fd15edda},
  author = {Jiacai Liu and Yingru Li and Yuqian Fu and Jiawei Wang and Qian Liu and Yu Shen},
  year = {2025},
  month = september,
}
```

## Overview

Rollout Correction addresses distribution mismatch between:
- **Rollout policy**: e.g., vLLM with BFloat16
- **Training policy**: e.g., FSDP with FP32

This mismatch can lead to biased gradient estimates and unstable training. Rollout Correction uses importance sampling (IS) weights and rejection sampling (RS) to correct these biases.

### Key Design Principle: Separation of IS Weights and Rejection Sampling

**Important**: As of 10/27/2025, the implementation separates two mechanisms:

1. **IS Weights** (`rollout_is_weights`): Ratios π_train/π_rollout with processing:
   - **Safety-bounded** to [exp(-20), exp(20)] ≈ [2e-9, 5e8] to prevent overflow:
     * Token level: Bounds per-token ratios
     * Sequence level: Bounds product of ratios (broadcast to all tokens in sequence)
     * Geometric level: Bounds geometric mean of ratios (broadcast to all tokens)
   - **Truncate mode**: Upper clamped via .clamp(max=upper_threshold)
   - **Mask mode**: Safety-bounded ratios preserved (no threshold clamping)
   - **All modes**: Zeroed at padding positions (response_mask == 0)
   - Used for policy gradient calculations

2. **Rejection Sampling** (`modified_response_mask`): Applied via response_mask
   - Mask mode: Excludes tokens/sequences with outlier IS ratios
   - Veto: Excludes sequences with catastrophic tokens
   - Used for loss aggregation (denominator calculation)

This separation ensures:
- ✅ Correct loss normalization (rejected samples excluded from denominator)
- ✅ Mode-specific weight processing (truncate: upper clamped, mask: safety-bounded only)
- ✅ Padding positions zeroed in weights (necessary for correct aggregation)
- ✅ Safety bounds always applied (prevent overflow in all modes)

## Configuration

```yaml
algorithm:
  rollout_correction:
    rollout_is: token                      # IS weights: "token", "sequence", or null
    rollout_is_threshold: 2.0              # Upper threshold for IS weights
    rollout_rs: null                       # Rejection sampling: "token", "sequence", "geometric", or null
    rollout_rs_threshold: null             # RS upper threshold (uses rollout_is_threshold if null)
    rollout_rs_threshold_lower: null       # RS lower threshold (auto-reciprocal if null)
    rollout_token_veto_threshold: null     # Per-token veto threshold (null = disabled)

# REQUIRED: Enable log prob calculation
actor_rollout_ref:
  rollout:
    calculate_log_probs: true
```

Key features:
- ✅ Three aggregation levels: token, sequence, geometric
- ✅ Two bounding modes: truncate, mask
- ✅ Dual threshold support (upper/lower)
- ✅ Veto mechanism for catastrophic outliers
- ✅ 30+ comprehensive metrics
- ✅ Log-space computation for numerical stability
- ✅ Memory-efficient implementation

## Files

### **Core Implementation**

- `verl/trainer/ppo/mismatch_helper.py` - Contains `compute_rollout_correction_and_rejection_mask()` and `compute_mismatch_metrics()`
- `verl/trainer/ppo/core_algos.py` - Rollout Correction integration with PPO
- `verl/workers/actor/dp_actor.py` - Metrics collection and logging

### **Configuration Files**

- `verl/trainer/config/algorithm.py` - Rollout Correction parameters in `AlgoConfig`
- `verl/workers/config/actor.py` - Rollout Correction parameters in `ActorConfig`
- `verl/trainer/config/actor/actor.yaml` - Rollout Correction configuration section
- `verl/trainer/config/ppo_trainer.yaml` - Algorithm config with Rollout Correction

### **Documentation**

- `docs/examples/config.rst` - Configuration parameter descriptions

### **Example Scripts**

- `recipe/dapo/run_dapo_qwen2.5_32b_rollout_corr.sh` - DAPO example with Rollout Correction
- `examples/rollout_correction/README.md` - Comprehensive usage guide
- `examples/rollout_correction/run_with_rollout_corr.sh` - Basic example

### **Tests**

- `tests/trainer/ppo/test_rollout_corr.py` - Unit tests
- `tests/trainer/ppo/test_rollout_corr_integration.py` - Integration tests

## Configuration Parameters

All parameters are under `algorithm.rollout_correction`:

### `rollout_is` (str or null)
Importance sampling weights aggregation level:
- `null` = No IS weights computed (metrics-only mode)
- `"token"`: Per-token IS weights ρ_t = π_train(t)/π_rollout(t)
  - Biased estimator, low variance
  - Recommended threshold: 1.5 - 5.0
- `"sequence"`: Per-sequence weight ρ_seq = ∏_t ρ_t
  - Unbiased estimator, high variance
  - Recommended threshold: 2.0 - 10.0

All IS weights are safety-bounded to [exp(-20), exp(20)] ≈ [2e-9, 5e8]

### `rollout_is_threshold` (float)
Upper threshold for IS weights. Default: `2.0`
- Used to clamp IS weights (not for rejection)
- Rejection is controlled by `rollout_rs` parameters

### `rollout_rs` (str or null)
Rejection sampling aggregation level:
- `null` = No rejection sampling
- `"token"`: Reject individual tokens with outlier ratios
- `"sequence"`: Reject entire sequences with outlier ratios
- `"geometric"`: Geometric mean aggregation for rejection
  - Recommended threshold: 1.0002 - 1.001

### `rollout_rs_threshold` (float or null)
Upper threshold for rejection sampling. Default: `null`
- If `null`, uses `rollout_is_threshold`
- Tokens/sequences with ratios > threshold are masked out

### `rollout_rs_threshold_lower` (float or null)
Lower threshold for rejection sampling. Default: `null`
- If `null`, uses reciprocal of upper threshold (1/upper)
- Tokens/sequences with ratios < threshold are masked out

### `rollout_token_veto_threshold` (float or null)
Per-token veto for catastrophic outliers. Default: `null`
- Checks **unclamped per-token ratios** before safety bounds
- If ANY token has ratio < threshold, entire sequence is rejected
- Independent of `rollout_is` and `rollout_rs` settings
- Recommended: `1e-4` to `1e-6` when enabled
- Example: `1e-4` catches tokens 10,000x less likely

### Summary: How IS Weights are Processed

The final IS weights go through multiple stages of processing:

**Stage 1: Safety Bound (All Modes)**
- Token level: `exp(clamp(log_ratio, -20, 20))` per token → bounds each token to [2e-9, 5e8]
- Sequence level: `exp(clamp(sum(log_ratio), -20, 20))` → bounds product to [2e-9, 5e8], broadcast to all tokens
- Geometric level: `exp(clamp(mean(log_ratio), -20, 20))` → bounds geometric mean to [2e-9, 5e8], broadcast to all tokens

**Stage 2: Threshold Processing (Mode-Dependent)**
- Truncate mode: `.clamp(max=upper_threshold)` → upper clamps weights to threshold
- Mask mode: No modification → weights remain as safety-bounded ratios

**Stage 3: Padding (All Modes)**
- `weights * response_mask` → zeros out padding positions

**Rejection Mechanisms (Modify response_mask, NOT weights)**
- Veto: Checks **unclamped per-token ratios** (before safety bound), rejects sequences via mask
- Outlier (mask mode only): Checks safety-bounded weights against [lower, upper], rejects via mask

## Operation Modes

The system uses **two independent mechanisms** that can be enabled separately or together:

1. **Importance Sampling (IS) weights**: Controlled by `rollout_is` parameter
2. **Rejection Sampling (RS)**: Controlled by `rollout_rs` parameter

### Mode Combinations

| `rollout_is` | `rollout_rs` | Behavior |
|--------------|--------------|----------|
| `null` | `null` | **Disabled**: No computation, no metrics, no rejection |
| `null` | `"token"` or `"sequence"` | **Rejection only**: Compute metrics, NO weight correction, YES rejection sampling |
| `"token"` or `"sequence"` | `null` | **IS weights only**: Weight correction enabled, NO rejection sampling |
| `"token"` or `"sequence"` | `"token"` or `"sequence"` | **Full correction**: Both weight correction and rejection sampling enabled |

### Key Insights

- ✅ You can use **rejection sampling alone** without IS weight correction (`rollout_is=null, rollout_rs="token"`)
- ✅ You can use **IS weights alone** without outlier rejection (`rollout_is="token", rollout_rs=null`)
- ✅ You can use **both together** (`rollout_is="token", rollout_rs="token"`)
- ✅ You can **monitor metrics only** without any correction by setting both to `null` but still providing rollout_log_probs

**Veto rejection** (if enabled via `rollout_token_veto_threshold`) is applied **independently** of IS and RS settings.

### Recommended Workflow

1. **Start with metrics only** to understand the mismatch:
   ```yaml
   algorithm:
     rollout_correction:
       rollout_is: null
       rollout_rs: null
   ```
   Monitor `mismatch/rollout_is_mean`, `mismatch/mismatch_kl` to assess distribution mismatch.

2. **Enable rejection sampling** if you see high outlier fractions:
   ```yaml
   algorithm:
     rollout_correction:
       rollout_is: null
       rollout_rs: token
       rollout_rs_threshold: 2.0
   ```
   This excludes outliers from training without modifying gradients.

3. **Enable full IS correction** once comfortable with metrics:
   ```yaml
   algorithm:
     rollout_correction:
       rollout_is: token
       rollout_is_threshold: 2.0
       rollout_rs: token
       rollout_rs_threshold: 2.0
   ```

## Usage

### Basic Setup

```yaml
algorithm:
  rollout_correction:
    rollout_is: token           # Enable IS weights at token level
    rollout_is_threshold: 2.0   # Threshold for IS weights
    rollout_rs: null            # No rejection sampling
    rollout_token_veto_threshold: null  # No veto

actor_rollout_ref:
  rollout:
    calculate_log_probs: true  # Required!
```

### Metrics

All metrics are prefixed with `mismatch/`. For example, `rollout_is_mean` appears as `mismatch/rollout_is_mean` in logs.

#### **Core IS Weight Metrics**

- **`rollout_is_mean`**: Mean importance sampling weight across all valid tokens
  - **Ideal value**: Close to 1.0 (indicates minimal distribution mismatch)
  - **Warning**: < 0.5 or > 2.0 suggests significant policy mismatch

- **`rollout_is_std`**: Standard deviation of IS weights
  - **Ideal value**: < 0.5 for stable training
  - **Warning**: > 1.0 indicates high variance, may need tighter thresholds

- **`rollout_is_min`**: Minimum IS weight observed
  - Shows the most underweighted token/sequence
  - For sequence/geometric: computed from unclamped log-space ratios (true minimum)
  - For token: computed from safety-bounded weights

- **`rollout_is_max`**: Maximum IS weight observed
  - Shows the most overweighted token/sequence
  - For sequence/geometric: computed from unclamped log-space ratios (true maximum before safety bound)
  - For token: computed from safety-bounded weights (before threshold clamping)
  - Compare with `rollout_is_threshold` to see truncation impact

#### **Effective Sample Size**

- **`rollout_is_eff_sample_size`**: Effective sample size after IS weighting
  - **Formula**: `1 / mean(weights²)` where weights are normalized
  - **Range**: 0.0 to 1.0 (as fraction of original batch)
  - **Ideal value**: > 0.5 (retaining at least 50% effective samples)
  - **Warning**: < 0.3 means high variance, losing too many effective samples

#### **Veto Mechanism Metrics**

- **`rollout_is_veto_fraction`**: Fraction of sequences rejected by veto mechanism
  - **Important**: Sequences are rejected via `response_mask=0`, NOT by modifying IS weights
  - **IS weights unchanged by veto**: Already processed by mode (truncate: clamped, mask: safety-bounded)
  - Veto checks **unclamped per-token ratios** π_train(t)/π_rollout(t) (true ratios before safety bound)
  - Detects catastrophic tokens (true ratio < veto_threshold, e.g., < 1e-4)
  - **Ideal value**: < 0.05 (less than 5% vetoed)
  - **Warning**: > 0.1 suggests policies are too different or numerical issues

- **`rollout_is_catastrophic_token_fraction`**: Fraction of tokens below veto threshold
  - Identifies problematic tokens before sequence-level veto is applied
  - Checks **unclamped per-token ratios** (true ratios, not safety-bounded)
  - Each catastrophic token causes its entire sequence to be rejected
  - **Warning**: > 0.01 indicates widespread distribution issues or numerical instability

#### **Threshold Exceedance Metrics**

- **`rollout_is_ratio_fraction_high`**: Fraction of weights exceeding upper threshold
  - Shows how often truncation/masking occurs on high end
  - For sequence/geometric: computed from unclamped log-space ratios (true exceedance)
  - For token: computed from safety-bounded weights (before threshold clamping)
  - **Ideal value**: < 0.1 (most weights within bounds)

- **`rollout_is_ratio_fraction_low`**: Fraction of weights below lower threshold
  - Shows how often masking occurs on low end (mask mode only)
  - For sequence/geometric: computed from unclamped log-space ratios (true exceedance)
  - For token: computed from safety-bounded weights
  - **Ideal value**: < 0.1

#### **Sequence-Level Metrics** (for sequence/geometric modes)

- **`rollout_is_seq_mean`**: Mean IS weight at sequence level
  - Should match `rollout_is_mean` for sequence-level aggregation

- **`rollout_is_seq_std`**: Standard deviation of sequence-level IS weights

- **`rollout_is_seq_min`**: Minimum sequence-level IS weight

- **`rollout_is_seq_max`**: Maximum sequence-level IS weight

- **`rollout_is_seq_max_deviation`**: Maximum absolute deviation from 1.0 at sequence level
  - **Ideal value**: < 1.0
  - Shows worst-case sequence mismatch

- **`rollout_is_seq_fraction_high`**: Fraction of sequences exceeding upper threshold

- **`rollout_is_seq_fraction_low`**: Fraction of sequences below lower threshold

#### **Masking Metrics** (mask mode only)

- **`rollout_is_masked_fraction`**: Fraction of tokens rejected via response_mask (mask mode only)
  - **Important**: Tokens are rejected by setting `response_mask=0`, NOT by modifying IS weights
  - **IS weights in mask mode**: Safety-bounded ratios preserved (no threshold clamping)
  - **Ideal value**: < 0.1 (less than 10% rejected)
  - **Warning**: > 0.3 means losing too much data

- **`rollout_is_seq_masked_fraction`**: Fraction of sequences with at least one rejected token
  - Shows sequence-level impact of rejection sampling
  - For token-level: sequence rejected if ANY token is outside [lower, upper]
  - For sequence-level: all tokens have same weight, so entire sequence rejected or accepted

#### **Distribution Mismatch Metrics** (Training vs Rollout Policy)

- **`mismatch_training_ppl`**: Perplexity of training policy (e.g., FSDP FP32)
  - **Formula**: `exp(-mean(log_probs))`
  - Lower is better (model is more confident)

- **`mismatch_rollout_ppl`**: Perplexity of rollout policy (e.g., vLLM BF16)
  - Should be close to `mismatch_training_ppl` if policies match well

- **`mismatch_ppl_ratio`**: Ratio of training PPL to rollout PPL
  - **Formula**: `exp(mean(log(training_ppl / rollout_ppl)))`
  - **Ideal value**: Close to 1.0
  - **Meaning**: > 1.0 means training is less confident than rollout

- **`mismatch_training_log_ppl`**: Log perplexity of training policy
  - Useful for identifying trends (linear scale)

- **`mismatch_rollout_log_ppl`**: Log perplexity of rollout policy

- **`mismatch_log_ppl_diff`**: Mean difference in log perplexities
  - **Formula**: `mean(log_ppl_rollout - log_ppl_training)`
  - **Ideal value**: Close to 0.0
  - Sign indicates which policy is more confident

- **`mismatch_log_ppl_abs_diff`**: Mean absolute log perplexity difference
  - Magnitude of mismatch regardless of direction

- **`mismatch_log_ppl_diff_max`**: Maximum log perplexity difference across sequences
  - Identifies worst-case sequence

- **`mismatch_log_ppl_diff_min`**: Minimum log perplexity difference across sequences

- **`mismatch_kl`**: KL divergence KL(π_rollout || π_training)
  - **Formula**: `mean(log_prob_rollout - log_prob_training)`
  - **Ideal value**: Close to 0.0 (policies match)
  - **Warning**: > 0.1 indicates significant mismatch
  - **Note**: Can be negative (rollout is less confident)

- **`mismatch_k3_kl`**: K3 KL estimator
  - **Formula**: `mean(exp(log_ratio) - log_ratio - 1)`
  - More stable for small KL values
  - Always non-negative

#### **Example: Accessing Metrics in Code**

```python
# Metrics are returned from compute_rollout_correction_and_rejection_mask
from verl.trainer.ppo.mismatch_helper import compute_rollout_correction_and_rejection_mask

# Returns 3 values (weights, modified_response_mask, metrics)
weights_proto, modified_response_mask, metrics = compute_rollout_correction_and_rejection_mask(
    old_log_prob=training_log_probs,      # from training policy
    rollout_log_prob=rollout_log_probs,   # from rollout policy
    response_mask=response_mask,
    rollout_is="token",  # Enable IS weights at token level
    rollout_is_threshold=2.0,
    rollout_rs="token",  # Enable rejection sampling at token level
    rollout_rs_threshold=2.0,
    rollout_rs_threshold_lower=0.5,
    rollout_token_veto_threshold=1e-4,  # Enable veto for catastrophic outliers
)

# Extract IS weights (processed, zeroed at padding)
is_weights = weights_proto.batch["rollout_is_weights"]

# IS weights processing (with IS enabled at token level):
# 1. Safety-bounded: exp(clamp(log_ratio, -20, 20)) per token
# 2. Zeroed at padding positions
# Note: Not threshold-clamped since we're using rejection sampling (rollout_rs)

# modified_response_mask has rejection applied (since rollout_rs="token"):
# 1. Outlier rejection: tokens outside [0.5, 2.0] masked to 0
# 2. Veto rejection: sequences with catastrophic tokens (ratio < 1e-4) masked to 0
# Note: Veto checks unclamped per-token ratios, not the safety-bounded weights

# All metrics have 'mismatch/' prefix
print(f"Mean IS weight: {metrics['mismatch/rollout_is_mean']:.3f}")
print(f"Effective sample size: {metrics['mismatch/rollout_is_eff_sample_size']:.3f}")
print(f"Veto fraction: {metrics['mismatch/rollout_is_veto_fraction']:.3f}")
print(f"Masked fraction: {metrics['mismatch/rollout_is_masked_fraction']:.3f}")
print(f"KL divergence: {metrics['mismatch/mismatch_kl']:.3f}")

# Check IS weights for valid tokens (non-padding)
valid_weights = is_weights[response_mask.bool()]
print(f"\n✓ IS weights min (valid tokens): {valid_weights.min():.4f}")
print(f"✓ IS weights max (valid tokens): {valid_weights.max():.4f}")
print(f"✓ All valid IS weights > 0: {(valid_weights > 0).all()}")

# Check rejection via response_mask
rejected_tokens = (response_mask == 1) & (modified_response_mask == 0)
print(f"\n✓ Rejected {rejected_tokens.sum()} tokens via response_mask")
print(f"✓ With rejection sampling (rollout_rs): tokens outside thresholds are masked")
print(f"✓ IS weights are always safety-bounded to [exp(-20), exp(20)] ≈ [2e-9, 5e8]")

# Check for warning conditions
if metrics['mismatch/rollout_is_mean'] < 0.5 or metrics['mismatch/rollout_is_mean'] > 2.0:
    print("⚠️  Warning: Mean IS weight far from 1.0, significant policy mismatch detected")

if metrics['mismatch/rollout_is_eff_sample_size'] < 0.3:
    print("⚠️  Warning: Low effective sample size, high variance in IS weights")

if metrics['mismatch/rollout_is_veto_fraction'] > 0.1:
    print("⚠️  Warning: High veto fraction, policies may be too different")
```

#### **Example: Monitoring Metrics During Training**

```python
# In your training loop
for epoch in range(num_epochs):
    for batch_idx, batch in enumerate(dataloader):
        # ... rollout phase ...

        # Compute IS weights and get metrics
        rollout_corr_config = config.algorithm.get("rollout_correction", None)
        if rollout_corr_config is not None:
            weights_proto, modified_response_mask, metrics = compute_rollout_correction_and_rejection_mask(
                old_log_prob=batch.old_log_prob,
                rollout_log_prob=batch.rollout_log_prob,
                response_mask=batch.response_mask,
                rollout_is=rollout_corr_config.get("rollout_is", None),
                rollout_is_threshold=rollout_corr_config.get("rollout_is_threshold", 2.0),
                rollout_rs=rollout_corr_config.get("rollout_rs", None),
                rollout_rs_threshold=rollout_corr_config.get("rollout_rs_threshold", None),
                rollout_rs_threshold_lower=rollout_corr_config.get("rollout_rs_threshold_lower", None),
                rollout_token_veto_threshold=rollout_corr_config.get("rollout_token_veto_threshold", None),
            )

        # Log to tensorboard/wandb
        for metric_name, metric_value in metrics.items():
            logger.log_scalar(metric_name, metric_value, step=global_step)

        # IMPORTANT: Update batch response_mask with rejection applied
        batch.response_mask = modified_response_mask

        # Use IS weights in training (always safety-bounded, zeroed at padding)
        is_weights = weights_proto.batch["rollout_is_weights"]
        # ... apply weights to policy gradient ...
```

#### **Example: Conditional Alerting Based on Metrics**

```python
def check_rollout_correction_health(metrics, config):
    """Check if Rollout Correction metrics indicate healthy training."""
    warnings = []

    # Check mean IS weight
    mean_weight = metrics['mismatch/rollout_is_mean']
    if mean_weight < 0.5 or mean_weight > 2.0:
        warnings.append(f"Mean IS weight {mean_weight:.3f} is far from 1.0")

    # Check effective sample size
    ess = metrics['mismatch/rollout_is_eff_sample_size']
    if ess < 0.3:
        warnings.append(f"Effective sample size {ess:.3f} is too low")

    # Check veto fraction
    veto_frac = metrics['mismatch/rollout_is_veto_fraction']
    if veto_frac > 0.1:
        warnings.append(f"Veto fraction {veto_frac:.3f} is too high")

    # Check variance
    std = metrics['mismatch/rollout_is_std']
    if std > 1.0:
        warnings.append(f"IS weight std {std:.3f} is too high")

    # Check KL divergence
    kl = metrics['mismatch/mismatch_kl']
    if abs(kl) > 0.1:
        warnings.append(f"KL divergence {kl:.3f} indicates significant mismatch")

    if warnings:
        print("⚠️  Rollout Correction Health Warnings:")
        for warning in warnings:
            print(f"  - {warning}")
        return False
    else:
        print("✅ Rollout Correction metrics look healthy")
        return True

# Use in training
_, _, metrics = compute_rollout_correction_and_rejection_mask(...)
is_healthy = check_rollout_correction_health(metrics, config)

if not is_healthy:
    # Consider adjusting config or investigating issues
    print("Consider:")
    print("  - Tightening rollout_is_threshold")
    print("  - Switching to geometric aggregation level")
    print("  - Checking if rollout and training policies are too different")
```

### Running Examples

Start with the basic token-level truncate configuration:
```bash
bash examples/rollout_correction/run_with_rollout_corr.sh
```

Monitor metrics for 1-2 epochs before adjusting parameters.

## Configuration Examples

### Example 1: IS Weights Only (Token Level)
```yaml
algorithm:
  rollout_correction:
    rollout_is: token
    rollout_is_threshold: 2.0
    rollout_rs: null  # No rejection sampling
```

### Example 2: Rejection Sampling Only (No IS Weights)
```yaml
algorithm:
  rollout_correction:
    rollout_is: null  # No IS weights
    rollout_rs: token
    rollout_rs_threshold: 2.0
    rollout_rs_threshold_lower: 0.5
```

### Example 3: Both IS and RS (Geometric RS)
```yaml
algorithm:
  rollout_correction:
    rollout_is: token
    rollout_is_threshold: 2.0
    rollout_rs: geometric
    rollout_rs_threshold: 1.0002
    rollout_rs_threshold_lower: 0.9998
```

### Example 4: Full Correction with Veto
```yaml
algorithm:
  rollout_correction:
    rollout_is: sequence
    rollout_is_threshold: 5.0
    rollout_rs: token
    rollout_rs_threshold: 5.0
    rollout_rs_threshold_lower: 0.2
    rollout_token_veto_threshold: 1e-4  # Veto catastrophic tokens
```

## Troubleshooting

### Issue: High variance in IS weights
**Symptoms:** `rollout_is_std` > 1.0, `rollout_is_eff_sample_size` < 0.3

**Solutions:**
1. Switch from `sequence` to `geometric` level
2. Tighten thresholds
3. Verify rollout and training aren't too different

### Issue: Too many sequences vetoed
**Symptoms:** `rollout_is_veto_fraction` > 0.1

**Solutions:**
1. Relax veto threshold in config:
   ```yaml
   algorithm:
     rollout_correction:
       rollout_token_veto_threshold: 1e-3
   ```
2. Check for numerical issues in log prob computation
3. Verify policies aren't completely different

### Issue: Mean IS weight far from 1.0
**Symptoms:** `rollout_is_mean` < 0.5 or > 2.0

**Solutions:**
1. Verify `calculate_log_probs=True` is set
2. Check rollout_log_probs are correctly passed
3. Check for systematic bias

### Debugging: Visualizing Metrics

**Example: Plot IS weight distribution**

```python
import matplotlib.pyplot as plt
import numpy as np

def plot_is_metrics(metrics_history):
    """Plot rollout IS metrics over training steps."""
    fig, axes = plt.subplots(2, 3, figsize=(15, 10))

    # Plot 1: Mean IS weight over time
    axes[0, 0].plot(metrics_history['mismatch/rollout_is_mean'])
    axes[0, 0].axhline(y=1.0, color='r', linestyle='--', label='Ideal')
    axes[0, 0].set_title('Mean IS Weight')
    axes[0, 0].set_xlabel('Step')
    axes[0, 0].legend()

    # Plot 2: Effective sample size
    axes[0, 1].plot(metrics_history['mismatch/rollout_is_eff_sample_size'])
    axes[0, 1].axhline(y=0.5, color='g', linestyle='--', label='Good')
    axes[0, 1].axhline(y=0.3, color='r', linestyle='--', label='Warning')
    axes[0, 1].set_title('Effective Sample Size')
    axes[0, 1].set_xlabel('Step')
    axes[0, 1].legend()

    # Plot 3: Veto fraction
    axes[0, 2].plot(metrics_history['mismatch/rollout_is_veto_fraction'])
    axes[0, 2].axhline(y=0.1, color='r', linestyle='--', label='Warning')
    axes[0, 2].set_title('Veto Fraction')
    axes[0, 2].set_xlabel('Step')
    axes[0, 2].legend()

    # Plot 4: KL divergence over time
    axes[1, 0].plot(metrics_history['mismatch/mismatch_kl'], label='KL')
    axes[1, 0].plot(metrics_history['mismatch/mismatch_k3_kl'], label='K3 KL')
    axes[1, 0].axhline(y=0, color='g', linestyle='--', alpha=0.3)
    axes[1, 0].set_title('KL Divergence')
    axes[1, 0].set_xlabel('Step')
    axes[1, 0].legend()

    # Plot 5: PPL ratio over time
    axes[1, 1].plot(metrics_history['mismatch/mismatch_ppl_ratio'])
    axes[1, 1].axhline(y=1.0, color='r', linestyle='--', label='Ideal')
    axes[1, 1].set_title('PPL Ratio (Training/Rollout)')
    axes[1, 1].set_xlabel('Step')
    axes[1, 1].legend()

    # Hide unused subplot
    axes[1, 2].axis('off')

    plt.tight_layout()
    plt.savefig('rollout_is_metrics.png', dpi=150)
    print("Saved plot to rollout_is_metrics.png")
```

**Example: Metric collection during training**

```python
# Collect metrics over time
metrics_history = {
    'mismatch/rollout_is_mean': [],
    'mismatch/rollout_is_eff_sample_size': [],
    'mismatch/rollout_is_veto_fraction': [],
    'mismatch/mismatch_kl': [],
    'mismatch/mismatch_k3_kl': [],
    'mismatch/mismatch_ppl_ratio': [],
}

# In training loop
for step in range(num_steps):
    # ... compute IS weights and rejection mask ...
    _, _, metrics = compute_rollout_correction_and_rejection_mask(...)

    # Store metrics
    for key in metrics_history.keys():
        if key in metrics:
            metrics_history[key].append(metrics[key])

    # Plot every 100 steps
    if step % 100 == 0:
        plot_is_metrics(metrics_history)
```

## Performance Impact

- **Memory overhead**: ~1% of model memory
- **Computational overhead**: 1-3% depending on level
- **Training stability**: Significantly improved when mismatch exists


## Testing

Run the test suite to verify everything works:

```bash
# Basic unit tests
python test_rollout_corr.py

# Integration tests (if pytest is available)
pytest tests/trainer/ppo/test_rollout_corr_integration.py -v
```

Expected output: All tests pass ✓

## Additional Resources

- **Implementation**: `verl/trainer/ppo/mismatch_helper.py`
- **Examples**: `examples/rollout_correction/`
- **DAPO Example**: `recipe/dapo/run_dapo_qwen2.5_32b_rollout_corr.sh`

## Summary

Rollout Correction provides:
- ✅ Robust handling of distribution mismatch
- ✅ Numerical stability
- ✅ Comprehensive metrics for monitoring
- ✅ Flexibility for different scenarios
- ✅ Memory-efficient computation

## References

- [When Speed Kills Stability: Demystifying RL Collapse from the Inference-Training Mismatch](https://yingru.notion.site/When-Speed-Kills-Stability-Demystifying-RL-Collapse-from-the-Inference-Training-Mismatch-271211a558b7808d8b12d403fd15edda)
- [Your Efficient RL Framework Secretly Brings You Off-Policy RL Training](https://fengyao.notion.site/off-policy-rl)