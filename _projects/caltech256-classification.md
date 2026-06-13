---
layout: project
title: Caltech256 Classification
slug: image-classification
summary: Image classification task comparing models with different fine-tuning, data augmentations, and quantization (QAT & PTQ).
---

# Caltech256 Classification

A systematic investigation into transfer learning for vision models — comparing architectures, fine-tuning strategies, data augmentation, and quantization across two datasets.

## Overview

The goal was simple but deliberate: understand *why* different training choices produce different results, not just which number comes out highest. Starting from linear probing to isolate raw feature extraction capability, then moving to full fine-tuning experiments where fine-tuning strategy, augmentation regime, and quantization method were varied independently.

Two datasets were used throughout. Caltech256 is a real-world dataset with varying image sizes, similar in distribution to ImageNet — the models' home territory. CIFAR-100 has smaller 32×32 images and was used primarily for linear probing to understand how resolution affects feature transferability. The contrast between the two turned out to be one of the more interesting findings.

## Methodology

### Phase 1 — Linear Probing

All model weights were frozen except the classification head, which was replaced to match the new class count. Trained for 5 epochs with no augmentation — just resize and normalisation. The point was to measure how much useful information each architecture had already encoded from ImageNet, without any task-specific adaptation.

Eleven architectures were evaluated spanning CNNs, pure transformers, hybrid transformer-CNN models, and MLP-based architectures.

| Model | Caltech256 Val Acc (%) | CIFAR-100 Val Acc (%) |
|---|---|---|
| EfficientNetV2-M | 93.00 | 64.78 |
| RegNetY-040 | 85.86 | 62.64 |
| ResNet50 | 82.09 | 51.89 |
| ConvNeXt Base | 82.30 | 51.71 |
| ResNet152 | 78.69 | 52.59 |
| PVT V2 B3 | 77.18 | 50.59 |
| ResNet18 | 75.73 | 44.71 |
| Swin Base | 73.75 | 26.85 |
| ViT Base | 14.47 | 18.77 |
| DeiT3 Base | 12.22 | 16.69 |
| MLP-Mixer | 6.46 | 8.96 |

**Key observation:** Pure transformer architectures (ViT, DeiT, MLP-Mixer) performed remarkably poorly as feature extractors — ViT achieved only 14.47% despite strong ImageNet training. This points to a fundamental issue: without spatial inductive bias, these models struggle to transfer positional understanding to a new dataset. Hybrid architectures (PVT, Swin) recovered significantly by adding convolutional operations, confirming that spatial locality matters for transfer learning, not just representational power.

The resolution gap between datasets also proved significant — CIFAR-100's 32×32 images degraded performance across all models, most severely for architectures that rely on patch-based tokenisation.

### Phase 2 — Fine-Tuning

Three fine-tuning strategies were implemented from scratch using a custom model wrapper built on [timm](https://github.com/huggingface/pytorch-image-models):

- **Full**: All layers trained simultaneously with a single base learning rate and cosine scheduler
- **Discriminative**: Layer-wise learning rates (5 rate groups per model stage) with a LambdaLR scheduler, higher rates for the classification head and progressively lower rates toward the early layers
- **Complete**: Sequential layer unfreezing — starts at the head, unfreezes one stage per epoch with a warmup-style lower rate, then switches to base rate once fully unfrozen. Designed to prevent catastrophic forgetting

Three augmentation strategies were also compared:

- **No augmentation**: Resize + normalisation only
- **Basic**: Colour jitter, rotation, random erase
- **Strong**: Basic augmentations at higher magnitude, plus Mixup and CutMix

Data loading was handled with [NVIDIA DALI](https://developer.nvidia.com/dali) for GPU-accelerated preprocessing, which showed significant caching benefits across repeated epochs on the same dataset.

Quantization was evaluated both during training (QAT) and post-training (PTQ), using the same model wrapper to keep comparisons clean.

---

## Results

### Fine-Tuning Strategy Impact

| Model | Full FP32 / QAT | Discriminative FP32 / QAT | Complete FP32 / QAT |
|---|---|---|---|
| EfficientNetV2-S | 86.30 / — | **93.47** / 92.19 | 93.10 / 93.10 |
| MobileNet Hybrid | 80.78 / 80.81 | **90.04** / 88.08 | 88.96 / 88.46 |
| ResNet50 | 88.52 / 89.33 | 89.43 / **90.00** | 89.23 / 89.16 |
| PVT V2 B1 | ~2% / — | 89.43 / 75.09 | **89.09** / 89.09 |
| RegNetY-008 | 81.15 / 79.33 | **87.31** / 87.65 | 87.18 / 87.21 |
| ResNet18 | 82.33 / 83.64 | 84.15 / 83.74 | **84.72** / 84.55 |
| MobileNet Conv | 72.13 / 77.11 | 81.02 / **84.99** | 82.90 / 82.90 |

The gap between Full and Discriminative/Complete was striking — RegNetY-008 improved by ~6%, EfficientNet by ~7%. Without a learning rate scheduler or layer-wise rates, the model overfits the training set quickly and loses generalisation. CNNs proved more resilient to this — ResNet50 achieved 88% even under the Full strategy — while transformers like PVT collapsed to ~2% accuracy without proper fine-tuning configuration.

### Data Augmentation Impact

| Model | No Aug (%) | Basic Aug (%) | Strong Aug (%) |
|---|---|---|---|
| MobileNet Hybrid | 87.48 | 89.16 | **90.04** |
| PVT V2 B1 | 87.31 | 87.21 | **89.09** |
| ResNet50 | 88.29 | 88.49 | **90.00** |
| ResNet18 | 82.63 | 82.00 | **84.55** |
| RegNetY-008 | **87.21** | 86.40 | 87.04 |
| MobileNet Conv | **84.35** | 84.05 | 83.98 |

Augmentation consistently helped larger, more expressive models (MobileNet Hybrid +2.56%, PVT +1.78%) but slightly hurt smaller ones. The pattern makes intuitive sense: a model that is already struggling to fit the dataset doesn't benefit from the training set being made harder. CutMix and Mixup add value only when the model has enough capacity to learn from partial or blended inputs.

### Quantization

QAT achieved ~73% size reduction across all models with minimal accuracy loss:

| Model | FP32 Size (MB) | QAT Size (MB) | Accuracy (QAT %) | ECE |
|---|---|---|---|---|
| EfficientNetV2-S | 78.11 | 21.33 | 93.10 | 0.155 |
| MobileNet Hybrid | 38.60 | 10.41 | 88.46 | 0.061 |
| PVT V2 B1 | 52.14 | 13.30 | 88.08 | 0.020 |
| RegNetY-008 | 21.68 | 5.78 | 87.21 | 0.034 |
| MobileNet Conv | 10.74 | 2.94 | 82.90 | 0.156 |

**RegNetY-008** is the most practical result: 87.21% accuracy in 5.78 MB with the lowest inference time outside MobileNet Conv, and well-calibrated (ECE 0.034). For constrained deployments this is the clear choice. EfficientNet achieves the best accuracy but at nearly 4× the size and ~3.5× the inference time.

---

## Key Findings

**Training strategy matters more than architecture.** A smaller model (RegNetY-008, 8M params) with proper discriminative fine-tuning outperformed its larger variant (RegNetY-040, 20M params) trained with a flat learning rate. Getting the training configuration right is not optional.

**Transformers are brittle without correct fine-tuning.** PVT collapsed to ~2% accuracy under the Full strategy — the same model reached 89% under Complete fine-tuning. The attention mechanism needs controlled, gradual adaptation. CNNs are more forgiving because their inductive bias gives them a useful starting point regardless of how aggressively they are trained.

**Spatial inductive bias is the deciding factor in transfer learning.** The linear probing results make this concrete: ViT and DeiT transferred almost nothing despite strong ImageNet pretraining. PVT's spatial reduction attention and Swin's shifted windows recovered performance significantly. For transfer learning tasks, architecture decisions made for ImageNet scale don't automatically translate to smaller or different-domain datasets.

**QAT is worth it.** ~73% size reduction with marginal accuracy loss (~0.2–0.4%) and improved calibration stability over PTQ. The extra training time is justified in any deployment-facing scenario.

---

## GPU Constraints

All experiments were run on an NVIDIA L4 GPU (22 GB). QAT roughly doubles memory usage vs standard training — ResNet50 went from ~10.5 GB to ~21.5 GB, requiring batch size reduction and activation checkpointing for EfficientNet. RegNetY-040 could not complete QAT even with checkpointing, so it was excluded from QAT comparisons.

---

## Repository

[GitHub Repo](https://github.com/JVarnica/Caltech256_classification)
