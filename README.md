# Segment This Thing: Reproduction and Reliability Evaluation

## Project website

This repository includes a dependency-free static website for the study. It is ready to deploy on Vercel from the repository root—no build command, output directory, or environment variables are required.

### Deploy to Vercel

1. Push this repository to GitHub.
2. In Vercel, choose **Add New → Project** and import the repository.
3. Leave **Framework Preset** as `Other` and keep the build/output fields empty.
4. Select **Deploy**.

For a local preview, run `python -m http.server 3000` in the repository root and open `http://localhost:3000`.

An external reproduction and evaluation of **Segment This Thing (STT-B)**, the CVPR 2025 point-prompted segmentation model based on foveated tokenization. The official pretrained model is evaluated on Oxford-IIIT Pet without retraining, then extended with prompt and degradation robustness, geometry analysis, confidence calibration, failure analysis, and a paired comparison with SAM ViT-B.

<p align="center">
  <a href="https://colab.research.google.com/github/gundeeps247/cv_dl/blob/main/main.ipynb"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open in Colab"></a>
  <a href="Report.pdf"><img src="https://img.shields.io/badge/report-PDF-EA4335" alt="Read the report"></a>
</p>

![Qualitative STT-B results, including successful masks and a catastrophic failure](cv_dl%20project/qualitative_results_v2.png)

## Highlights

- The final primary evaluation successfully processed **499 of 500** fixed samples.
- STT-B achieved **0.6070 mean IoU** and **0.7149 mean Dice** on Oxford-IIIT Pet.
- Practical end-to-end latency was **93.0 ms/image**, or approximately **11 FPS**, on an NVIDIA Tesla T4.
- On 100 paired images, SAM ViT-B improved mean IoU from **0.6306 to 0.6717**, while STT-B was approximately **5.7× faster**.
- The predicted-IoU score was only moderately correlated with actual IoU (`r = 0.518`); **22 high-confidence catastrophic failures** were observed.
- Low Boundary F1 and large area/perimeter errors show that useful object localization does not guarantee accurate contours.

## Primary results

The primary protocol uses the deepest-interior foreground pixel as a single positive point. Images with a shorter side below 1,280 pixels are enlarged before inference, and predictions are mapped back to original resolution before scoring.

| Metric | Mean | 95% bootstrap CI |
|---|---:|---:|
| IoU | 0.6070 | 0.5844–0.6280 |
| Dice | 0.7149 | 0.6918–0.7362 |
| Precision | 0.8550 | 0.8464–0.8641 |
| Recall | 0.7226 | 0.6943–0.7487 |
| Boundary F1 | 0.0952 | 0.0881–0.1024 |
| End-to-end latency | 0.0930 s | 0.0917–0.0943 s |

Additional means: **28.3 ms** model-only latency, **0.397 GB** peak allocated GPU memory, **34.85%** area error, and **27.06%** perimeter error.

Precision is higher than recall, so STT-B more often omits part of the object than includes unrelated foreground. The much lower Boundary F1 indicates that region overlap and contour accuracy should be interpreted separately.

## STT-B vs SAM ViT-B

Both models receive the same 100 images, resized inputs, and deepest-interior prompts. SAM timing includes image embedding (`set_image`) and prompt decoding, making this a single-image end-to-end comparison.

| Metric | STT-B | SAM ViT-B |
|---|---:|---:|
| Mean IoU | 0.6306 | **0.6717** |
| Mean Dice | 0.7368 | **0.7686** |
| Precision | **0.8472** | 0.7952 |
| Recall | 0.7589 | **0.8685** |
| Boundary F1 | **0.1007** | 0.0636 |
| End-to-end latency | **0.0768 s** | 0.4400 s |
| Throughput | **13.54 FPS** | 2.28 FPS |

STT-B records the higher IoU on 46 paired images and SAM on 54. The mean paired IoU difference, STT minus SAM, is `−0.0411` (95% CI `−0.0819` to `−0.0017`). The result is best understood as a quality–speed frontier: SAM has moderately better average overlap, while STT responds much faster.

<table>
  <tr>
    <td><img src="cv_dl%20project/stt_vs_sam_iou.png" alt="Mean IoU comparison between STT-B and SAM ViT-B"></td>
    <td><img src="cv_dl%20project/stt_vs_sam_runtime.png" alt="End-to-end latency comparison between STT-B and SAM ViT-B"></td>
  </tr>
</table>

## Robustness and reliability

Prompt location materially changes segmentation quality. The foreground centroid produces the highest mean IoU on the 100-image ablation, while the pre-specified deepest-interior prompt gives the highest precision.

| Prompt | Mean IoU | Mean precision |
|---|---:|---:|
| Centroid | **0.6498** | 0.8116 |
| Deepest interior | 0.6161 | **0.8465** |
| Random interior | 0.5318 | 0.8212 |
| Near boundary | 0.5317 | 0.7866 |

![Mean IoU under four point-prompt strategies](cv_dl%20project/prompt_robustness_iou.png)

The same subset was evaluated after four controlled image transformations.

| Condition | Mean IoU | Change from original |
|---|---:|---:|
| Dark | **0.6286** | +0.0125 |
| Original | 0.6161 | — |
| Low resolution | 0.6079 | −0.0082 |
| Blur | 0.6004 | −0.0157 |
| Bright | 0.5744 | −0.0417 |

Darkening happens to improve the subset mean, but this non-monotonic response is data-dependent and is not evidence that darker images are generally easier.

For reliability analysis, a catastrophic failure is defined as actual IoU below `0.25`; a confident failure also requires predicted IoU of at least `0.70`.

| Reliability statistic | Result |
|---|---:|
| Predicted-IoU / actual-IoU correlation | 0.5182 |
| Catastrophic failures | 66 / 499 (13.2%) |
| Confident catastrophic failures | 22 / 499 (4.4%) |

![STT predicted IoU versus actual IoU](cv_dl%20project/confidence_calibration.png)

The model score is directionally useful, but the broad scatter and high-confidence low-IoU cases mean that it should not be treated as a per-image correctness guarantee.

## Methodology

1. Load the 3,669-image Oxford-IIIT Pet test split and convert trimap label `1` to foreground.
2. Shuffle with seed `42`, request 500 primary samples, and reject masks with fewer than 100 foreground pixels.
3. Generate one deepest-interior positive point using a distance transform.
4. Upscale the shorter image side to at least 1,280 pixels, preserving point coordinates.
5. Run the official STT-B predictor, choose the candidate with the highest predicted IoU, reconstruct the foveated mask, and map it back to original resolution.
6. Measure region metrics, two-pixel-tolerance Boundary F1, model and end-to-end latency, GPU allocation, and mask-derived geometry.
7. Run fixed 100-image prompt, degradation, and paired SAM experiments.
8. Estimate confidence intervals with 2,000 bootstrap resamples and export plots, CSV files, and a JSON summary.

## Run the notebook

The notebook is designed for Google Colab and installs its own dependencies.

1. Open [`main.ipynb`](main.ipynb) in Colab.
2. Select a GPU runtime; the reported run used an NVIDIA Tesla T4.
3. Keep `PROTOCOL = "final"` for the 500/100 design, or use `"sanity"` for a 20-image pipeline check.
4. Run all cells from a fresh runtime.

The notebook downloads Oxford-IIIT Pet, the official STT-B checkpoint, and the SAM ViT-B checkpoint. Internet access, a CUDA-capable runtime, and several gigabytes of temporary storage are recommended.

Generated artifacts are written to:

```text
/content/stt_project_outputs_final_500_100
```

This includes the primary and robustness tables, paired model deltas, bootstrap intervals, confidence and failure summaries, qualitative figures, and `final_report_summary.json`.

## Repository contents

```text
.
├── main.ipynb                                     # Executed Colab experiment
├── Report.pdf                                     # Project report
├── cv_dl project/                                 # Exported figures and metrics
└── README.md
```

## Limitations

- This is an external evaluation, not an exact reproduction of the paper's benchmark suite.
- The Oxford trimap conversion excludes its boundary label from foreground, affecting contour-sensitive scores.
- Inputs are enlarged by **3.79× on average**, which can alter fine edge information.
- Only one positive point is used; corrective positive and negative clicks are not studied.
- The paired SAM baseline contains 100 images rather than all 499 valid primary samples.
- Boundary F1 depends on the notebook's fixed two-pixel tolerance.

## References

- Tanner Schmidt and Richard Newcombe, [“Segment This Thing: Foveated Tokenization for Efficient Point-Prompted Segmentation,”](https://openaccess.thecvf.com/content/CVPR2025/html/Schmidt_Segment_This_Thing_Foveated_Tokenization_for_Efficient_Point-Prompted_Segmentation_CVPR_2025_paper.html) CVPR 2025.
- Alexander Kirillov et al., [“Segment Anything,”](https://openaccess.thecvf.com/content/ICCV2023/html/Kirillov_Segment_Anything_ICCV_2023_paper.html) ICCV 2023.
- Omkar M. Parkhi et al., [“Cats and Dogs,”]([https://www.robots.ox.ac.uk/~vgg/publications/2012/Parkhi12a/](https://ieeexplore.ieee.org/document/6248092)) CVPR 2012.
