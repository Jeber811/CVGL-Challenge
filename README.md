# CVGL-Challenge

WRIVA Cross-View Geo-Localization Challenge
Leaderboard Rank: 15th Place Globally | Final Score: 107.36

Overview
This repository contains the inference and submission pipeline developed for the 2026 IARPA WRIVA Cross-View Geo-Localization (CVGL) Challenge. The objective of the system is to map sequential ground-level camera frames to overhead Maxar satellite GeoTIFFs by predicting precise geographic metadata (latitude, longitude, heading, pitch).

The pipeline is built on a customized DINOv3-based Johns Hopkins University (JHU) architecture and features significant mathematical, algorithmic, and infrastructure upgrades to ensure highly accurate, crash-resilient evaluation on Codabench.

Core Technical Interventions
1. Algorithmic & Mathematical Upgrades
Softmax Spatial Centroid Aggregation: Eliminated a ~32-meter grid quantization penalty inherent to the baseline model. Implemented a 50% sliding window overlap (128x128 chips with a 64px stride) and applied a softmax algorithm to the top overlapping predictions. This dynamically interpolates a score-weighted, continuous geographic coordinate rather than rigidly snapping to a pre-defined grid center.

Dynamic CRS Reprojection: Engineered an automatic coordinate detection function utilizing rasterio and pyproj. It identifies proprietary metric localizations in satellite GeoTIFFs and dynamically transforms them back to standard WGS84 (EPSG:4326) lat/lon coordinates on the fly.

Smart Variance Filter: Optimized inference speed by computing pixel variance on satellite chips before they reach the GPU, automatically bypassing blank or low-information tiles (variance < 10) to save processing bandwidth.

Corrupted Image Guardrails: Implemented exception-handling fallbacks that detect unreadable ground images and safely default to the geographic center of the testing site, preventing catastrophic crashes and strict missing-file penalties.

2. Infrastructure & Systems Engineering
Local NVMe Caching Bypass: Re-architected the pipeline to clone neural network weights and Python modules directly to the virtual machine's local NVMe drive prior to execution. This completely isolated the Python interpreter from severe network disconnects (Errno 107) caused by the FUSE driver buckling under heavy read loads.

Stateful Resume Architecture: Transformed standard batch processing into a stateful, frame-by-frame persistent tracking system. A "skip-ahead" scanner verifies existing JSON outputs on a persistent cloud drive before loading images into the GPU, allowing 10-hour execution sequences to be safely interrupted and instantly resumed across different servers without losing computational progress.

3. Automated Packaging & Data Routing
Codabench Validation Compliance: Built a custom JSON writer and shutil.make_archive compiler to distribute thousands of individual {lat, lon, heading, pitch} payloads into their respective parent site folders.

Domain Shift Mapping: Wrote conditional string parsing logic to navigate strict naming schemas. The pipeline correctly routes Domain A test sets (siteA01 through siteA11) alongside distinct, out-of-distribution datasets (siteM02), dynamically repairing truncated dictionary keys and resolving FileNotFoundErrors during data loading.

Tech Stack
Languages: Python

Deep Learning: PyTorch, Hugging Face Hub (DINOv3)

Geospatial Processing: Rasterio (Geotransforms), PyProj

Systems & Automation: OS, Shutil, Subprocess, JSON

Environment: Google Colab (T4 GPU), Codabench Evaluation

Execution Flow
Environment Initialization: Sets expansion configurations (PYTORCH_CUDA_ALLOC_CONF), injects API tokens, and dynamically rewrites infer_pipeline.json with target dataset paths.

Staging & Isolation: Verifies remote mounts and migrates model classes and best.pt checkpoint weights to local block storage.

Stateful Inference Loop: Scans target directories, bypasses existing processed frames, and slides a 128x128 window across Maxar satellite imagery to extract 2048-dimensional feature embeddings.

Cosine Similarity Matching: Ranks similarity scores between ground chips and satellite frames, applying the spatial aggregation algorithm to determine final coordinates.

Audit & Package: Verifies the final prediction count against raw input file totals, constructs the nested Codabench directory tree, and compiles the final submission .zip.
