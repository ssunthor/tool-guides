# Video Generation Processor (VGP) - Design Document

**Version:** 0.0.1
**Date:** 2026-02-01
**Category:** Processor (P)
**Acronym:** VGP
**Status:** Draft
**Author:** Gemini AI / Martin Kecka

---

## 1. Introduction & Purpose

The **Video Generation Processor (VGP)** is a specialized AI-driven module designed to automate the end-to-end production of high-quality corporate video assets.

It serves as the "Media Factory" for the SERA ecosystem, transforming structured input (scripts, assets, style guides) into polished video content suitable for internal communications, training, and product launches.

Unlike simple text-to-video wrappers, VGP implements a **Hybrid Production Workflow** that orchestrates multiple generative models (Text-to-Video, Image-to-Video, Text-to-Speech) and combines them with traditional assembly techniques (FFmpeg) to produce long-form, coherent content that would be impossible to generate in a single AI pass.

---

## 2. Architecture

VGP is architected as a Python-based CLI application that interacts with the SERA Shared Infrastructure and the AI Factory.

## 2. Architecture

VGP is architected as a sequential, state-managed pipeline designed to maximize visual fidelity on limited VRAM hardware (24GB). Unlike real-time systems, VGP prioritizes generation quality over inference speed.

### 2.1 Core Components

* **VGP Core (Controller):** The main logic engine. It parses `project.yaml` and executes the "Production Plan."
* **ModelContextManager (The Garbage Collector):** A critical component for the 24GB VRAM constraint. It ensures **mutual exclusivity** of models.
    * *Logic:* Before `VisualGenerator` loads the Video Model, `ModelContextManager` forces the unload of `AudioGenerator` and runs `torch.cuda.empty_cache()`.
    * *Strategy:* Implements "CPU Offloading" for large models, keeping weights in system RAM and streaming to VRAM only during inference.
* **AudioGenerator:** modular interface for Text-to-Speech (TTS).
* **VisualGenerator:** The high-fidelity rendering engine.
    * **Unified Model Approach:** Uses a single, large-parameter model (Wan 2.2 14B) for *both* T2V and I2V to ensure color grading and grain consistency across the entire video.
* **Upscaler:** A post-processing neural network (Real-ESRGAN) that takes the raw 720p AI output and upscales it to 4k for sharp text readability.
* **Assembler:** A wrapper around `ffmpeg-python` that stitches clips, performs cross-dissolve transitions, and ensures audio sync.

---

## 3. The Quality-First Workflow ⚙️

Generating a broadcast-ready 3-minute video requires a "Sequential High-Fidelity" strategy.

1.  **Project Initialization:** Parse `project.yaml` and validate assets.
2.  **Audio Synthesis (Voiceover):** Generate all audio tracks first.
    * *Output:* Precise duration timestamps (e.g., Scene 1 = 4.2s).
3.  **Visual Generation (The Heavy Lift):**
    * **Context Switch:** Load Wan 2.2 (14B) with CPU Offloading.
    * **Generation Loop:** Iterate through scenes one by one.
        * *Drafting:* Generate the clip at native resolution (720x1280 or 1280x720).
        * *Drift Correction:* If the video is shorter than the audio, generate a "tail extension" or freeze-frame.
4.  **AI Upscaling (The Polish):**
    * **Context Switch:** Unload Video Model -> Load Upscaler (Real-ESRGAN).
    * **Process:** Frame-by-frame upscale of all generated clips from 720p to 4k (3840x2160). This is essential for ensuring the dashboard text in the CAP UI screenshots remains crisp.
5.  **Assembly & Post-Processing:**
    * **Stitching:** FFmpeg combines the 4k clips.
    * **Transitions:** Apply 0.5s cross-dissolves between clips to remove "jump cuts."
    * **Audio Mix:** Duck background music volume during voiceover segments.

---

## 4. Data Schema (`project.yaml`)

VGP uses a declarative YAML schema to define video projects. This allows for version control of video production.

```yaml
project:
  name: "CAP_Launch_Video"
  output_resolution: "1080p" # or 4k
  
  style:
    key_color: "#4ED869" # RGB(78, 216, 105)
    font: "Roboto"
    logo_path: "/assets/images/logos/gtt_logo_white.png"
    tone: "High-tech, clean, corporate, professional"
  
  audio:
    provider: "elevenlabs"
    voice_id: "Callum" # Professional Narration
    music_track: "/assets/audio/music/corporate_tech_bg.mp3"
    music_volume: 0.15

  scenes:
    - id: "intro"
      type: "t2v"
      text: "Today, we’re excited to introduce CAP — the CV Analysis Processor — our AI powered engine designed to transform how we evaluate talent at GTT."
      prompt: "Cinematic 3D animation of abstract digital data streams connecting in a dark void. Glowing emerald green lines (color #4ED869) forming complex network structures. High contrast, sleek, corporate tech aesthetic, 4k resolution, slow smooth motion."
      overlay_text: "Introducing CAP"

    - id: "section_1_dashboard"
      type: "i2v"
      source_image: "screenshots/cap_dashboard.png"
      text: "CAP analyzes resumes, job descriptions, and interview transcripts to give fast, reliable, and consistent assessments. It evaluates alignment, credibility, skill recency, career trajectory, stability, and more."
      prompt: "Slow, smooth camera pan moving horizontally across a software dashboard screen. High resolution, sharp text, professional software interface presentation. Subtle glowing green accents."
      
    # ... additional scenes ...
```

## 5. Reference Implementation: CAP Launch Video

This section defines the specific assets, script mapping, and AI prompt strategy required to generate the "Introduction to CAP" video requested by the business. This reference implementation serves as the validation test case for the VGP system.

### 5.1 Prerequisites & Asset Setup
The following high-resolution screenshots must be captured from the running CAP application and placed in the project's asset directory (`/assets/video_processor/cap_launch/screenshots/`):

1.  `dashboard.png`: The main statistics dashboard showing the "Processing Speed" and "Open Positions" cards.
2.  `candidate_detail.png`: A detailed candidate report view showing the "Credibility Score" and "Badges" section.
3.  `analysis_list.png`: The main analysis list view with several candidates marked as "Strong Candidate" or "High Potential Fit".
4.  `job_modal.png`: The "Edit Job Position" modal showing the "Hard Criteria" and "Key Criteria" input fields.

### 5.2 Scene Definitions & Prompt Strategy

The following prompts have been engineered for the **Wan 2.2 (I2V)** and **LTX-Video (T2V)** models to ensure a consistent "High-tech, clean, corporate" aesthetic using the brand color **#4ED869**.

#### **INTRO (0:00–0:10)**
* **Visual Type:** Text-to-Video (T2V)
* **Audio Script:** "Today, we’re excited to introduce CAP — the CV Analysis Processor — our AI powered engine designed to transform how we evaluate talent at GTT."
* **Generation Prompt:**
    > Cinematic 3D animation of abstract digital data streams connecting in a dark void. Glowing emerald green lines (color #4ED869) forming complex network structures. High contrast, sleek, corporate tech aesthetic, 4k resolution, slow smooth motion.

#### **SECTION 1 — What CAP Does (0:10–0:35)**
* **Visual Type:** Image-to-Video (I2V)
* **Source Asset:** `dashboard.png`
* **Audio Script:** "CAP analyzes resumes, job descriptions, and interview transcripts to give fast, reliable, and consistent assessments. It evaluates alignment, credibility, skill recency, career trajectory, stability, and more."
* **Generation Prompt:**
    > Slow, smooth camera pan moving horizontally across a software dashboard screen. High resolution, sharp text, professional software interface presentation. Subtle glowing green accents.

#### **SECTION 2 — What Makes CAP Different (0:35–1:05)**
* **Visual Type:** Image-to-Video (I2V)
* **Source Asset:** `candidate_detail.png`
* **Audio Script:** "CAP performs a multi stage, forensic level review — checking verified experience, role alignment, skill relevance, promotions, gaps, inconsistencies, and red flags."
* **Generation Prompt:**
    > Slow camera zoom in towards the "Credibility Score" and "Badges" section of a software interface. Cinematic depth of field blurring the edges, keeping the center metrics in sharp focus. High tech analysis feeling.

#### **SECTION 3 — How CAP Works (1:05–2:30)**
* **Scene 3.1 (Safety)**
    * **Type:** T2V
    * **Prompt:** `Abstract visualization of a digital shield verification. A glowing green scanning beam passing over a digital document, verifying security. Dark background, cyber-security aesthetic, precise and clean.`
* **Scene 3.2 (Trajectory)**
    * **Type:** T2V
    * **Prompt:** `3D infographic line graph rising upwards, glowing bright green against a dark background. Representing career growth and stability. Clean corporate motion graphics style.`
* **Scene 3.3 (Credibility)**
    * **Type:** T2V
    * **Prompt:** `Digital magnifying glass analyzing lines of code and text documentation. Searching for inconsistencies. Matrix-style data flow in background, subtle green tint. High-tech forensic review.`
* **Scene 3.4 (Scoring)**
    * **Type:** I2V (Asset: `analysis_list.png`)
    * **Prompt:** `Camera panning down a list of candidate results. Stopping on a "Strong Candidate" badge with a high score. The score glows slightly.`

#### **SECTION 4 — Why Accurate Job Descriptions Matter (2:30–3:00)**
* **Visual Type:** Image-to-Video (I2V)
* **Source Asset:** `job_modal.png`
* **Audio Script:** "CAP is only as precise as the job description you provide. Clear must haves, dealbreakers, skills, and expectations dramatically improve accuracy and candidate match quality."
* **Generation Prompt:**
    > Static shot of a job description form. A virtual cursor types "Must have Python experience" into a text field. Soft lighting, professional software demo style.

#### **SECTION 5 & CLOSING (3:00–End)**
* **Visual Type:** Text-to-Video (T2V)
* **Audio Script:** "CAP will elevate how we hire — but it starts with accurate job descriptions. When you provide clarity, CAP delivers precision."
* **Generation Prompt:**
    > Cinematic slow motion shot of a diverse group of corporate professionals standing in a modern glass office, looking confident and happy. Ideally wearing business casual. Subtle green lighting accents in the background. High quality stock footage style.

## 6. Technical Stack & Hardware Optimization

### 6.1 Model Selection (Quality Focused)

Since inference time is not a constraint, we discard "distilled" or "turbo" models in favor of maximum parameter count models to ensure accurate prompt adherence and visual coherence.

* **Unified Video Model: Wan 2.2 (14B Parameters)**
    * **Role:** Handles both Image-to-Video (I2V) and Text-to-Video (T2V).
    * **Why:** Wan 2.2 14B currently holds the SOTA (State of the Art) for motion smoothness and physical correctness. Using one model prevents jarring stylistic shifts between scenes.
    * **Configuration:**
        * `precision`: `fp16` (Half Precision) - No 8-bit quantization artifacts.
        * `offload_device`: `cpu` - This allows the 30GB model to run on the 24GB L4 GPU by swapping layers dynamically.

* **Upscaling: Real-ESRGAN**
    * **Model:** `RealESRGAN_x4plus` (trained on general images) or `RealESRGAN_x4plus_anime_6B` (often better for sharp UI/Vectors).
    * **Why:** Native AI video is soft. This restores the crisp edges of the CAP dashboard text.

### 6.2 Hardware Strategy (NVIDIA L4 - 24GB)

To run this pipeline without Out-Of-Memory (OOM) errors, the `vgp.py` controller enforces a strict **Single-Tenant GPU Policy**:

1.  **TTS Phase:** VRAM Usage ~2GB. (Video models must be unloaded).
2.  **Video Phase:** VRAM Usage ~23GB (Saturated).
    * Uses `accelerate` library for CPU offloading.
    * Inference speed: ~2.5s per frame (vs 0.5s native).
    * Total render time for 3min video: ~3-4 hours (Acceptable per requirements).
3.  **Upscale Phase:** VRAM Usage ~4-8GB. (Video models must be unloaded).

### 6.3 Dependencies

* **Diffusers:** `git+https://github.com/huggingface/diffusers.git` (Bleeding edge required for Wan 2.2).
* **Torch:** `torch>=2.4.0` with CUDA 12.4+.
* **Video IO:** `decord` (Efficient frame reading for upscaling).


### Next Steps for Implementation

1.  **Scaffold the Project:** Create the directory structure (`/ai/video_gen_processor/`) and the `project.yaml` file based on the schema above.
2.  **Asset Collection:** Take the 4 required screenshots from your running CAP instance and place them in the `/assets` folder.
3.  **Model Setup:** Decide whether to run **Wan 2.2** locally on your H100s (recommended for cost/privacy) or use an external API for the first prototype.