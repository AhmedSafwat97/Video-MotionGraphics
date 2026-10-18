# **Motion Pipeline**

A free/open-source pipeline for producing MP4 motion graphics videos with code, works with any AI agent:

**Design/style + content + voiceover → animated scenes + SFX + synchronization → `out/video.mp4`**

Tools: **Remotion** (React) for scenes and rendering, **Whisper** for analyzing the voiceover and its timings, **FFmpeg** for audio and video.

---

## **1. Requirements (one time)**

| **Tool** | **Version** | **Test** |
| --- | --- | --- |
| Node.js | 18+ | `node -v` |
| Python | 3.9+ | `python --version` |
| FFmpeg | any recent | `ffmpeg -version` |

After that, from the project folder:

bash

```
npm install
pip install -r requirements.txt
./scripts/make_placeholders.sh     # SFX and a demo audio so the project runs
./scripts/doctor.sh                # checks everything and tells you what's missing
```

> The first Whisper run downloads the model. Default is `small`. For higher accuracy: `WHISPER_MODEL=medium` (about 1.5GB).
> 

---

## **2. Quick start with the agent**

1. Unzip the file and open the agent **from inside the project folder** (Claude Code / Gemini CLI / Codex CLI / Cursor / Copilot in Agent mode).
2. Add the inputs (Section 3).
3. Tell the agent:

text

```
Read START_HERE.md and execute it.
```

The agent must be able to run commands in the terminal and read and write files. If not, it will write the code for you and you run the commands yourself.

---

## **3. Inputs: the two modes**

### **Mode A: Sample style only (the agent designs the scenes)**

- Put one or more images in `samples/`. These are a **visual identity reference only** (colors, fonts, vector shapes, glow/stroke/radius, overall mood). The agent will **not** copy their layout.
- Give the agent the content: video idea, script, or storyboard (in chat or in a `brief.md` file). And if you have a ready voiceover, that's enough, and the agent transcribes it with Whisper and splits it into scenes.

### **Mode B: Scene designs ready (the agent implements and animates them)**

- Put the design of **each scene** in `designs/` with ordered names: `01_hook.png`, `02_problem.png`, ...
- Each file = **the final look** of the scene (the last frame after the animation ends).
- Best to send **SVG or a Figma export** instead of PNG: 100% match and separate elements ready for animation.

If both exist: `designs/` are the scenes, and `samples/` is additional style for anything missing.

### **The voiceover**

Put it in `public/voiceover.mp3`. Without it there's no synchronization (you can design and preview the scenes first, and record later).

### **The script text (optional but improves accuracy)**

Write the voiceover text in `script.md` (one line per sentence). When present:

- **forced alignment** (`stable-ts`): timing is computed against the words of the text itself, so it bypasses Whisper transcription errors in dialect and technical terms (RabbitMQ, queue, worker).
- If alignment isn't available, the text is sent to Whisper as an `initial_prompt` to improve recognition.
- The agent picks more accurate keywords for the start of each scene.
- `python scripts/transcribe.py --check` compares the text with the recording and warns if they differ (if you recorded with different wording, update the text or delete `script.md`).

**Important:** the text alone without the audio has no timing. Timing always comes from the recorded audio.

### **Linking elements to words (cues)**

In `storyboard.json` for any scene:

json

```
"cues": [{"word": "الرسالة", "target": "msg", "action": "enter", "lead": 0.15}],
"sfx":  [{"name": "ding", "word": "وصلت"}]
```

`build_timeline.py` computes their timing from the audio, and the scene receives them as the `cues` prop (use `cueFrame` from `src/cues.ts`).

### **Video dimensions**

In `storyboard.json`:

json

```
{ "fps": 30, "width": 1080, "height": 1920 }
```

| **Platform** | **Dimensions** |
| --- | --- |
| Reels / TikTok / Shorts | 1080×1920 |
| YouTube | 1920×1080 |
| Instagram post | 1080×1350 |
| Square | 1080×1080 |

Set the dimensions **before** building the scenes and don't change them afterwards (the layout is built on them). If image names contain `@2x` then their actual dimensions are half.

---

## **4. What happens (the flow)**

**Mode A:** environment check ← style extraction (🛑 confirmation) ← scene splitting (🛑 approval) ← building scenes ← audio ← render ← review.

**Mode B:** environment check ← style extraction from the designs ← decomposition into layers ← rebuilding with code to an exact match ← comparing the last frame with the design ← animating from context ← audio ← render ← review.

Stop points (the agent asks you): dimensions if unspecified, confirming the extracted style (especially fonts), approving the scene split, and the presence of the voiceover.

### **Synchronization**

1. `scripts/transcribe.py` ← Whisper ← `work/voiceover.json` (timing of every word).
2. `scripts/build_timeline.py` reads `storyboard.json` and matches each scene's `keyword` (the first word spoken in the audio for the scene) with the words ← writes `src/timeline.json`.
3. Remotion places each scene and its sounds in its position from `timeline.json`.

If a keyword isn't found (especially with dialect), the script warns you. Put `"start": <seconds>` manually in the scene in `storyboard.json`.

---

## **5. Audio**

- **The voiceover:** `public/voiceover.mp3`.
- **SFX:** mp3 files in `public/sfx/`. The existing ones are demo (`ding`, `whoosh`, `tick`) generated by `make_placeholders.sh`. **Replace them** with real files under the same names or with new names you put in `storyboard.json`.
- Free sources: freesound.org, Pixabay, Mixkit. **Check the license** (some require attribution).
- Sounds only for important moments: each scene in `storyboard.json` has:

json

```
"sfx": [{"name": "ding", "offset": 2.3, "volume": 0.5}]
```

`offset` in seconds from the start of the scene.

---

## **6. Commands**

| **Purpose** | **Command** |
| --- | --- |
| Environment check | `./scripts/doctor.sh` |
| Live preview | `npm run studio` |
| Single frame for review | `npx remotion still Main out/check.png --frame=60` |
| Full flow (Whisper ← timeline ← high-quality render) | `./scripts/run_all.sh` |
| Render only | `npm run render` (or `npm run ultra` for 2x, slow) |
| Higher quality (2x) | `npx remotion render Main out/video.mp4 --scale=2 --crf 16` (very heavy on weak machines) |
| Analyze sample colors | `python scripts/analyze_design.py samples/*.png` |
| Convert an image to SVG (for complex ones) | `python scripts/trace_vectors.py in.png out.svg` |

Output: `out/video.mp4` (H.264).

---

## **6.5 Quality and time**

Defaults: **high quality at reasonable time** (H.264, crf 16, 1080p, 30fps). Nothing requires a GPU.

**How time is determined:** rendering spends its time drawing each frame in the browser (on the CPU), not in encoding. So duration is determined by (number of frames × cost per frame). Frame cost increases with blur, large shadows, and the number of elements. Changing `crf` won't make much difference to time.

| **Purpose** | **Command** |
| --- | --- |
| Measure full render time on your machine (one minute) | `./scripts/estimate_time.sh` |
| Final render (default) | `./scripts/run_all.sh` |
| Fast draft for review | `./scripts/run_all.sh draft` |
| Frame range | `npx remotion render Main out/part.mp4 --frames=0-120` |
| Control parallelism | `CONCURRENCY=2 ./scripts/run_all.sh` |
| Optional, slow: 2x | `./scripts/run_all.sh ultra` |
| Optional, more accurate audio transcription | `WHISPER_MODEL=medium ./scripts/run_all.sh` (or `large`) |

What actually reduces time:

- Light scenes (shadows/glow used sparingly, SVG instead of many elements). The agent is committed to this in `AGENTS.md`.
- 30fps (60fps doubles the time).
- Drafts for review and the final once.
- Whisper `small` + `script.md` (alignment compensates for accuracy). It's skipped if `work/voiceover.json` is newer than the audio and text (`SKIP_TRANSCRIBE=0` to force it).
- Laptop on the charger, and close heavy programs.

## **7. Project structure**

text

```
AGENTS.md            rules and instructions for any agent
START_HERE.md        the full prompt (the agent executes it from start to finish)
MOTION.md            motion and timing rules (durations, easing, linking to words)
CLAUDE.md GEMINI.md .cursorrules .github/copilot-instructions.md   (all point to AGENTS.md)
storyboard.json      scenes + dimensions + keywords + SFX
samples/             reference style (Mode A)
designs/             ready scene designs (Mode B)
brief.md             (optional) idea/storyboard written as text
script.md            (optional) the exact voiceover text (improves timing accuracy)
public/voiceover.mp3 the voiceover
public/sfx/          the effects
src/tokens.ts        colors and fonts (the single source of style)
src/components/      reusable elements
src/scenes/          the scenes + index.ts (registering them)
src/timeline.json    generated automatically, don't edit manually
src/Main.tsx Root.tsx   assembling scenes and sounds
scripts/             transcribe, build_timeline, analyze_design, trace_vectors, run_all, doctor
work/ out/           intermediate files and output
```

> The existing components and scenes (`Hook`, `PhoneGrid`, `NotifCard`, `Statement`...) are ready technical examples that work as a starting point. The agent will replace them according to your style and content.
> 

---

## **8. Limits you should know**

- **Fonts:** automatic matching is approximate. For a commercial or custom font, put its file in `public/` and tell the agent.
- **Vectors from images:** geometric shapes and text are made with code at high accuracy. Complex drawings (characters, details) come out approximate from tracing, so a ready SVG is better.
- **Colors:** extraction is roughly accurate, review it.
- **Arabic and Egyptian dialect in Whisper:** `medium` is good, and `large` is more accurate. Review the important words.
- **No "3D" quality:** the project is 2D/Motion Graphics. If you want 3D, you can use `@remotion/three` or Blender, but that goes outside the flat style.

---

## **9. Troubleshooting**

| **Problem** | **Solution** |
| --- | --- |
| `ffmpeg` / `ffprobe` not found | Install it and make sure it's in the PATH |
| Whisper is slow | Use `small`, or a GPU |
| Keyword not found | Set `start` manually in `storyboard.json` |
| Audio out of sync | Check `src/timeline.json` and re-run `build_timeline.py` |
| Warning that the text differs from the recording | Update `script.md` or delete it and rely on Whisper |
| `stable-ts` not installed | `pip install stable-ts`, and if unavailable it falls back to Whisper automatically |
| The agent isn't following instructions | Tell it explicitly: "Read START_HERE.md and execute it" |
| Render is slow or the machine hangs | Reduce `CONCURRENCY`, and review with `draft` and `remotion still` before the final |
| Elements scattered after changing dimensions | Layout is in pixels. Ask the agent to rearrange it for the new dimensions |

---

## **10. Licenses**

- **Remotion:** free for individuals and small companies, paid for larger companies. Check the current terms before commercial use: https://www.remotion.dev/license
- **Whisper, FFmpeg, vtracer:** open source.
- **SFX and fonts:** check the license of each file and font.
