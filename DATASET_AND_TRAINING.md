# E-RayZer: Dataset and Training Information

## What is This Repository About?

**E-RayZer** is a **self-supervised 3D reconstruction model** designed for **spatial visual pre-training**. It is a 3D vision model that:

- **Predicts camera poses** from multi-view images without requiring ground truth camera parameters
- **Reconstructs 3D scene geometry** represented as 3D Gaussian splats
- Uses **transformer-based architecture** to process multi-view images
- Performs **novel view synthesis** by rendering the scene from arbitrary viewpoints

### Key Capabilities
- Takes multiple RGB images of a scene from different viewpoints
- Automatically estimates the camera poses (position and orientation) for each image
- Creates a 3D representation of the scene using Gaussian splatting
- Can render novel views of the scene from any camera position
- Works in a **self-supervised** manner, learning from multi-view consistency

### Architecture Overview
- **Image Tokenizer**: Converts input images into patch tokens (256×256 images, 16×16 patches)
- **Transformer Encoder**: 16-layer encoder for geometric understanding (768-dim, 64-dim heads)
- **Transformer Decoder**: 8-layer decoder for Gaussian prediction
- **Gaussian Renderer**: Differentiable rendering using gsplat with intrinsics-gradient support

---

## Training Data Requirements

### Dataset

The model is trained on the **DL3DV dataset**, as specified in the configuration file (`config/erayzer.yaml`).

### How Many Images/Views Are Needed?

#### For Training:
From the training configuration in `config/erayzer.yaml` (check the file for current values):

- **Total views per scene**: 10 views
- **Input views**: 5 views (used to predict the scene)
- **Target views**: 5 views (used for supervision)
- **Batch size**: 24 scenes per GPU
- **Training iterations**: ~152K forward-backward passes

> **Note**: These values are from the default configuration. Always refer to `config/erayzer.yaml` for the most current training parameters.

The training process uses:
- **Covisibility-based view selection** with curriculum learning
- **Frame distance curriculum**: Starts with nearby views (min: 1.0, max: 1.0) and progresses to more distant views (min: 1.0, max: 0.5) over 86,000 iterations
- **Random split**: Views are randomly split into input and target sets
- **Domain randomization** to improve generalization

#### For Inference:
- **Recommended**: ~10 multi-view RGB images of the same scene
- **Minimum**: Can work with fewer images (the engine automatically pads/repeats views if you provide less than 10)
- **Format**: The model processes exactly 10 views internally; if fewer are provided, the last view is repeated

---

## Data Format

### Image Requirements

#### Format Specifications:
- **File formats**: PNG, JPG, JPEG, or WebP
- **Color space**: RGB (3 channels)
- **Recommended resolution**: 256×256 pixels
- **Bit depth**: 8-bit per channel

#### Multi-View Capture:
The images should be:
1. **Multiple views of the same scene** captured from different camera positions
2. **Overlapping content** - images should have sufficient overlap to establish correspondences
3. **Diverse viewpoints** - covering different angles around the object/scene
4. **Consistent lighting** - preferably captured under similar lighting conditions

### Training Data Structure

Based on the example files in `examples/example_0_dl3dv_000053/`:
```
scene_name/
├── view_000_000000.png  (256×256 RGB)
├── view_001_000021.png
├── view_002_000042.png
├── view_003_000064.png
├── view_004_000096.png
└── ...
```

The naming convention appears to be: `view_{view_index}_{frame_index}.png`

### Configuration Parameters

Key training parameters from `config/erayzer.yaml`:

```yaml
model:
  image_tokenizer:
    image_size: 256          # Input image size
    patch_size: 16           # Patch size for tokenization
  target_image:
    height: 256
    width: 256
  transformer:
    d: 768                   # Embedding dimension
    d_head: 64              # Attention head dimension
    encoder_n_layer: 16     # Encoder depth
    decoder_n_layer: 8      # Decoder depth
  gaussians:
    sh_degree: 3            # Spherical harmonics degree
    n_gaussians: 1          # Gaussians per pixel

training:
  dataset: dl3dv            # DL3DV dataset
  batch_size_per_gpu: 24
  num_views: 10
  num_input_views: 5
  num_target_views: 5
  lr: 0.0004
  weight_decay: 0.05
  warmup: 3000
```

### Data Preprocessing

The model uses:
- **Center cropping** (`center_crop: true`)
- **Square cropping** (`square_crop: true`)
- **Distance normalization** to 2.0 units (`normalize_distance_to: 2.0`)
- **Relative pose representation** (`use_rel_pose: true`)
- **Domain randomization** for robustness (`domain_randomization: true`)

### Camera Parameters

- The model **does not require ground truth camera intrinsics** for training (`use_gt_intrinsics: false`)
- Predicts camera poses in a **self-supervised manner** from multi-view consistency
- Uses a **near plane** of 0.2 and depth range configured in the model

---

## Pre-trained Models

Two pre-trained models are available:

1. **erayzer_multi.pt** (Recommended): 
   - Trained on multiple datasets for better generalization
   - Default model for the demo
   - Best choice for general-purpose 3D reconstruction on diverse scenes
   
2. **erayzer_dl3dv.pt**: 
   - Trained exclusively on DL3DV dataset
   - May perform better on scenes similar to DL3DV data distribution
   - Use if you're working specifically with DL3DV-style data

Models are licensed under Adobe Research License (non-commercial use only). Download automatically via Gradio or manually from [Hugging Face](https://huggingface.co/qitaoz/E-RayZer/tree/main/checkpoints).

---

## Inference Example

For inference with the Gradio app:
```bash
python gradio_app.py \
    --config config/erayzer.yaml \
    --ckpt checkpoints/erayzer_multi.pt \
    --device cuda:0 \
    --output-dir outputs
```

Upload ~10 multi-view RGB images, and the system will:
1. Predict camera poses for all views
2. Reconstruct 3D geometry as Gaussian splats
3. Generate novel view renders
4. Export outputs:
   - **point_cloud.glb**: 3D scene as a colored point cloud in GLB format (includes camera frustums for visualization)
   - **render_video.mp4**: Turntable video showing novel view synthesis
   - **pred_view_*.png**: Rendered images from predicted viewpoints
   - **ZIP archive**: Complete bundle of all outputs

---

## Summary

- **Purpose**: Self-supervised 3D reconstruction and camera pose estimation
- **Training data**: 10-view multi-view image sets from DL3DV dataset
- **Inference data**: ~10 RGB images (256×256) of the same scene from different viewpoints
- **Format**: Standard RGB images (PNG/JPG), no camera parameters required
- **Output**: Predicted camera poses, 3D point cloud (GLB), novel view renders, and turntable video
