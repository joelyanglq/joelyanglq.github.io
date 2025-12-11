---
title: "Training-free Image Reconstruction via Visual
 Autoregressive Models"
collection: projects
tags: [AI, Generative Model]
permalink: /projects/2024-02-17-paper-title-number-4
excerpt: 'We benchmark AR type model performance on inverse problem solving, comparing to diffusion-based solver.'
date: 2025-12-10
venue: 'GitHub Journal of Bugs'
paperurl: '/files/ECE598_Final_Report.pdf'
citation: 'Your Name, You. (2024). &quot;Paper Title Number 3.&quot; <i>GitHub Journal of Bugs</i>. 1(3).'
header:
  teaser: var_inv.png
---

![](/images/var_inv.png)
Recent advances in diffusion and score-based generative models have dramatically improved the quality of learned priors for solving ill-posed imaging inverse problems. However, diffusion-based solvers typically require hundreds of function evaluations, leading to slow inference and high computational cost, and may hallucinate structures inconsistent with the measurements. In parallel, Visual Autoregressive Modeling (VAR) has emerged as a new paradigm for image generation that performs coarse-to-fine ``next-scale prediction'' and achieves diffusion-level image quality with significantly faster sampling. In this work, we take a benchmarking perspective and explore how well an off-the-shelf VAR model can serve as a training-free prior for image reconstruction. We formulate inverse problems in the latent token space of VAR and propose a two-stage inference scheme that combines hard token injection at coarse scales with measurement-gradient-guided logit refinement at finer scales. Our method requires no task-specific retraining and is evaluated on ImageNet-1k under five degradations: masking (inpainting), Gaussian blur, motion blur, nonuniform blur, and $$4\times$$ super-resolution. Quantitatively, our VAR-based solver achieves PSNR around $$19.7$$ dB and SSIM around $$0.85$$ across these tasks, which is substantially worse in PSNR than diffusion- and plug-and-play-based methods (typically $$26$$--$$30$$ dB) but competitive in SSIM, while running in a small fraction of their runtime. These results highlight both the current limitations of discrete visual autoregressive priors for inverse problems and their potential as fast, training-free learned priors. We view this work as an initial benchmark and analysis of VAR-style models for imaging inverse problems, rather than a new state-of-the-art method.
