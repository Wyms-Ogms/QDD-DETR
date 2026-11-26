# QDD-DETR

> **QDD-DETR: Query-Detail Decoupled Sparse Decoding with Pre-Pooling Stem Retrieval for Aerial Small Object Detection**

## Release plan and data availability

QDD-DETR forms a core part of the second author’s ongoing thesis. To comply with our institution’s requirements on thesis originality and intellectual property, **we will open-source the full QDD-DETR source code and pretrained models in this repository immediately after the second author completes the thesis defense.**

The experiments use two publicly available aerial detection benchmarks. Images and annotations are available from the dataset authors. [VisDrone2019](https://github.com/VisDrone/VisDrone-Dataset) · [SODA-A](https://shaunyuan22.github.io/SODA/)

## Query-detail decoupled decoding (QDD)

Small objects photographed from the air may occupy only a few pixels. On VisDrone2019 validation, an initial RT-DETR-R50 query center falls within two pixels of the bounding box for 90.8% of targets smaller than 16 pixels. Final predictions correctly match 42.7% of targets in that size group. This gap motivated us to give the decoder access to finer spatial detail during box refinement.

1. **Select queries on P3–P5.** The encoder scores positions on these three semantic maps and passes the top-scoring queries to the decoder. At 640 × 640 input resolution, it scores 8,400 positions.
2. **Retrieve fine detail during box refinement.** In the ResNet-50-vd backbone, a lateral branch projects the stride-4 C2 feature into a 128-channel P2 map. Stem-level detail recall (SDR) taps the 64-channel, stride-2 stem response before max-pooling to supply S2. Each query can sample both maps through deformable cross-attention.
3. **Project after sampling.** Deferred value projection (DVP) aggregates the sampled features, then applies the projection once per query, head and detail level. An extra channel filled with ones is sampled alongside the features. At the image boundary, its interpolated value scales the bias by the fraction of the sampling weights that falls inside the image. This preserves the dense projection result up to floating-point rounding.

## Training strategy

### Initialize the added levels with pretrained-weight grafting (PWG)

For COCO-pretrained DEIM-R50, copy the shared P3–P5 parameters from the pretrained detector. Initialize the P2 and S2 input projections to produce zero output. Set the new attention-logit weights to zero and their biases to −2. This gives the detail levels a small initial attention share while keeping the decoder output close to the pretrained prediction.

### Fine-tune with the detector's original objective

The detector learns when and where to sample P2 and S2 through its existing matching procedure, classification loss and box-regression losses. The matched DEIM-R50 experiments use 900 queries and a batch size of four on one GPU. On VisDrone2019, train for 60 epochs with 640 × 640 inputs. On SODA-A, train for 36 epochs on 800 × 800 slices, using repeat-factor sampling for both the baseline and QDD.

## Where is QDD in the code?

These paths refer to the reproduction package scheduled for release after the thesis defense.

| Component | File in the planned release |
| --- | --- |
| DEIM-R50 implementation | `vendor/DEIM/engine/deim/multi_octave.py` contains the backbone feature taps (`PResNetMO`), decoder (`RTDETRTransformerv2MO`) and deferred attention calculation (`lazy_level_attn`). |
| DEIM-D-FINE implementation | `vendor/DEIM/engine/deim/dfine_mo.py` adapts the early feature taps and decoder to the HGNetv2 backbone. |
| Pretrained-weight grafting | `deim_port/make_coco_mo_ckpt.py` expands the checkpoint. `deim_port/make_id_ckpt.py` sets the added attention-logit biases. |
| Training and evaluation | `run.py` selects the recipe from `manifest/runs.json` and dispatches training or evaluation. The main VisDrone configuration is `deim_port/configs/deim_r50vd_mo_p2s2zgid_q900_coco_visdrone.yml`. |
| Architecture diagram | `render/make_framework_pptx.py` draws the feature paths and sampling operations in Figure 2. |
| Error analysis and detection examples | `experiments/analyze_errors.py` classifies small-target errors for Figure 5. `experiments/qualitative.py` generates detection panels for Figure 6. |

## Evaluation

The table compares COCO-pretrained DEIM-R50 with and without QDD. Both models use 900 queries. The scores come from final-epoch checkpoints at seed 0. On SODA-A, both runs use repeat-factor sampling and allow up to 300 detections per slice.

| Validation metric | DEIM-R50 | DEIM-R50 + QDD | Gain (points) |
| --- | ---: | ---: | ---: |
| VisDrone2019 AP | 30.17 | 32.80 | +2.63 |
| VisDrone2019 small-object AP | 20.21 | 23.83 | +3.62 |
| SODA-A AP | 44.68 | 45.55 | +0.87 |

At 640 × 640 input resolution, the two detail levels add 3.1 GFLOPs to the 900-query DEIM-R50. In a separate timing experiment with RT-DETR-R50, adding QDD increased batch-one inference latency.

## Effect on small-object detections

- **Final box refinement (Figure 5).** For VisDrone2019 validation targets smaller than 16 pixels, adding QDD to the 640-input RT-DETR-R50 raised the correct-match rate from 42.7% to 48.0%. Localization errors fell from 11.3% to 8.8%, and misses from 37.2% to 34.1%.
- **Crowded scenes (Figure 6).** At a confidence threshold of 0.5, QDD found 64 of 110 annotated objects in the nighttime crowd crop, compared with 32 for the baseline. In the daytime road crop, the counts were 57 and 42 out of 84. The two crops were selected to illustrate gains in correct matches.

## Acknowledgements

We thank the [DEIM](https://github.com/Intellindust-AI-Lab/DEIM) and [Ultralytics](https://github.com/ultralytics/ultralytics) teams for their detection frameworks, and the VisDrone and SODA teams for making their benchmarks available.
