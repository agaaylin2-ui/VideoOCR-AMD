# DirectML desktop performance on the RX 9070 XT

## Reproducing the benchmark

Use the desktop Python 3.12 runtime, its verified det/rec model directory, and a checkout of the source. The profiler runs the normal video-to-SRT export on DirectML adapter 0. Supply a new output directory for each run.

```powershell
& $python amd_experiment/profile_video.py $video $models results-base
& $python amd_experiment/profile_video.py $video $models results-tail --start 20:16 --end 22:16 --workers 2 --batch 4
```

For a baseline, copy the same profiler into the released Preview 5 checkout. Keep the crop, language, video, model files and settings identical, and run one export at a time. Compare the SRT files including timestamps. Use `--lang ch` or `--lang chinese_cht` for the Chinese exports. Worker and batch overrides apply only to the benchmark. The batch option controls file prefetch and the OCR pipeline input batch; the recognizer retains its own configured line batch. OpenCV thread overrides apply to the benchmark process and its children.

Each run writes an SRT, the normal provider evidence, per-process JSON lines and a timing summary. Stage 1 extends from export entry to detection backend entry. Stage 2 includes detection, redundancy analysis and preparing the recognition inputs. Stage 3 includes worker startup, OCR, ordered delivery and shutdown. Finalization includes frame mapping, temporary-file removal, assembly and output. Costs recorded at the model runner boundary include synchronization and transfers. Worker seconds overlap across workers, as do JPEG reads with inference. Receipts record when the parent consumes the first result of each 32-image chunk.

## Worker and batch experiment

Final two minutes of the 22:16 bilingual episode, English, 637 OCR images. Forced worker counts apply to both passes, including the short detection pass. Times below are single runs in seconds. All eight output files matched byte for byte.

| Workers | Input batch | OpenCV threads | Stage 3 | Total |
|---:|---:|---|---:|---:|
| 1 | 4 | default | 15.63 | 19.73 |
| 2 | 4 | default | 10.75 | 16.72 |
| 3 | 4 | default | 14.90 | 21.70 |
| 1 | 4 | 1 | 18.25 | 22.36 |
| 2 | 4 | 1 | 11.80 | 17.73 |
| 3 | 4 | 1 | 12.38 | 18.37 |
| 3 | 2 | 1 | 10.68 | 16.64 |
| 3 | 8 | 1 | 11.18 | 17.14 |

## Where the time goes

The full Preview 5 run produced 7,321 Stage 3 JPEGs, all 720 × 100. The first quarter averaged 2.55 recognized regions per image and the last quarter 15.85. The slowest chunks began at images 6304, 6336 and 6368, with 1,709, 1,668 and 1,676 recognized regions. They took 15.24, 14.65 and 14.24 seconds. The median chunk took 1.43 seconds. Timing only the final two minutes misses part of this dense sequence.

The three workers completed 80, 76 and 73 chunks. Summed across them, the recognition runner used 407 seconds, JPEG reading 63, recognition postprocessing 46, preprocessing 15, detection inference 14 and detection postprocessing 6. Mean sampled worker RSS increased from 524 to 738 MiB across those quarters. The density increase and recognition timings explain expensive chunks. These measurements do not establish a memory leak or isolate GPU transfer time.

Ordered delivery delayed a completed chunk by up to 2.41 seconds. The executor already schedules queued chunks while the parent waits. This explains pauses in visible progress without establishing that out-of-order collection would shorten the export. Provider samples confirmed DirectML detection and recognition, with sampled CPU Slice and Concat operations in recognition. Measuring individual transfers would require a separate device trace.

Stage 2 already removed 5,117 redundant frames. Its boxes come from stitched detection grids. Stage 3 detects again on individually repacked JPEGs. Equivalent boxes, scores and SRTs from reusing grid detections have not been demonstrated. The baseline spent 1.26 seconds removing the working directory and 0.11 seconds assembling the SRT.

## Full episode and short control

Input was AX_spiritual_realm_ep_13_eng.mp4 (22:16), English, default settings, DirectML adapter 0. The baseline source tree matched release tag v1.6.0-amd-preview.5. The candidate caps the OCR pass at two processes; detection can still use three. Single-worker short exports take the same application path.

| Input and version | Detection / OCR workers | Stage 1 s | Stage 2 s | Stage 3 s | Final s | Total s | OCR images/s |
|---|---|---:|---:|---:|---:|---:|---:|
| Full episode, Preview 5 | 3 / 3 | 15.56 | 13.23 | 175.65 | 1.80 | 206.24 | 41.68 |
| Full episode, candidate | 3 / 2 | 15.41 | 11.36 | 171.98 | 1.66 | 200.41 | 42.57 |
| 04:02–04:11, Preview 5 (47 images) | 1 / 1 | 0.79 | 1.86 | 2.84 | 0.02 | 5.51 | 16.56 |
| Same excerpt, candidate | 1 / 1 | 0.75 | 1.59 | 2.22 | 0.02 | 4.57 | 21.20 |

The full-episode SRT matched byte for byte (SHA-256 67bc6bfb1105e8cb49b4c176bf393158b6cf5b6a8689ef64be047697d9bcd8c3). The nine-second outputs matched the reference SRTs in English, Simplified Chinese and Traditional Chinese. A historical English full-episode export had different cues in the credits; the quality comparison above uses the paired runs on the same machine and settings. This benchmark produced 7,321 recognition images rather than the originally observed 7,562.

With two workers, full-episode median/max chunk times were 0.99/8.25 seconds and maximum completion-to-delivery delay was 0.88 seconds. The workers handled 115 and 114 chunks. Process shutdown took 0.47 seconds in Stage 3. Mean last-quarter worker RSS increased from 738 MiB per worker with three to 1,238 MiB per worker with two. Memory availability limits are still applied. The recognizer receives contiguous CPU NumPy inputs and returns outputs through ONNX Runtime session.run for CPU decoding. The measurements do not separate host-device copy costs from inference.

The full-episode Stage 3 improvement was 2.1% in one paired run. Setup/cache and run-to-run variation affect the totals, as illustrated by the single-worker control. Verify on additional episodes before generalizing the result.
