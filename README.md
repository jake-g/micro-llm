# Micro LLMs [wip]

by [jake-g](https://github.com/jake-g)

Project root for working with various forks and experiemnts for Micro LLM project. See `.gitmodules` for included repositories.

`micro-llm` is use-case agnostic: it holds the `ESP32-S3` inference port, a generic tiny-model trainer, and pinned upstream sources. Domain tasks, data sources, and prompts live in consumer projects that import the trainer as a package; nothing here depends on a consumer. Sizes tagged `est.` are computed from published configs; claims not confirmed from a primary source are tagged `unverified`.

### Layout

- [`esp32-llm/`](esp32-llm/README.md): ESP32-S3 inference firmware (submodule, [jake-g/esp32-llm](https://github.com/jake-g/esp32-llm), edited here). Verify a plugged-in board with `python esp32-llm/scripts/verify_board.py --port /dev/cu.usbmodem2101`.
- [`tinyllm-training/`](tinyllm-training/README.md): generic installable Python package `tinyllm` (`pip install -e micro-llm/tinyllm-training`) that turns tidy time-series rows into prompt/target lines, trains `llama2.c` models on the PC GPU, and exports v0 `.bin` checkpoints for `esp32-llm`. Tasks and sources plug in through a registry; consumer projects own their tasks, data, runs, and result docs. Candidate model sizes and board fit are in its [Model Configs](tinyllm-training/README.md#model-configs).
- [`tiny_llama_voice.ipynb`](tiny_llama_voice.ipynb): `llama2.c` plus `SAM` voice Colab notebook (see [SAM Voice Research](#sam-voice-research)).
- [`third_party/llama2.c`](third_party/llama2.c/README.md) and [`third_party/SAM`](third_party/SAM/README.md): pinned upstream submodules, read-only. Consumer firmware may compile `third_party/SAM/src` directly.

### Contents

- [Measured Baseline](#measured-baseline)
- [Memory Budget](#memory-budget)
- [Pretrained Models](#pretrained-models)
- [Community MCU Builds](#community-mcu-builds)
- [Quantization, Runtimes](#quantization-runtimes)
- [Speedup Techniques](#speedup-techniques)
- [SAM Voice Research](#sam-voice-research)
- [Project Links](#project-links)

---

## Measured Baseline

- **Measured**: `stories260K` (dim 64, 5 layers, vocab 512, fp32, 1.05 MB) runs at **20.0 tok/s** on the Waveshare `ESP32-S3FH4R2` (4 MB flash, 2 MB quad PSRAM, `/dev/cu.usbmodem2101`), built with ESP-IDF v6.1 and `esp-dsp` 1.8.2; it ran at 18.6 tok/s on v5.5.
- **Upstream**: [DaveBben/esp32-llm](https://github.com/DaveBben/esp32-llm) reports 19.13 tok/s at 240 MHz CPU, 80 MHz PSRAM, using `dsps_dotprod_f32_aes3` and a Core 1 matmul task.
- **Bottleneck**: 260K params is about 0.5 MFLOP per token, so fixed overheads (per-matmul task sync, fp32 PSRAM traffic, sampling) dominate over MACs.
- **Format**: The port loads v0 fp32 `.bin`; [`runq.c`](third_party/llama2.c/runq.c) adds v2 `Q8_0` (`export.py --version 2`, group size 64, see [int8 quantization](third_party/llama2.c/README.md#int8-quantization)).
- **Sub-Repository Sync**: [`third_party/llama2.c`](third_party/llama2.c/README.md) (`master` at `350e04f`) and [`third_party/SAM`](third_party/SAM/README.md) (`master` at `5f54f9f`, SDL-free fork of `s-macke/SAM`) track `origin/master` cleanly.
- **Robustness**: [`esp32-llm/main/llm.c`](esp32-llm/main/llm.c) runs a boot self-test against golden greedy output (`make verify` via [`esp32-llm/scripts/verify_board.py`](esp32-llm/scripts/verify_board.py)).
- **Trained Checkpoint**: An example consumer `s260k` export passes the `llm.c` legacy v0 size check ([verification status](tinyllm-training/README.md#verification-status)).

---

## Memory Budget

| Resource | ESP32-S3 N16R8 | ESP32-P4 | LLM Use |
| :--- | :--- | :--- | :--- |
| CPU | 2x LX7, 240 MHz | 2x RISC-V, 400 MHz | Matmul |
| SIMD | PIE v1 (int8, int16) | PIE v2 (int8, int16) | Quantized GEMV |
| SRAM | 512 KB | 768 KB (`unverified`) | Activations, hot KV |
| PSRAM | 8 MB octal, 80 MHz | Up to 32 MB hex, 200 MHz (`unverified`) | Weights, KV |
| Flash | 16 MB quad | Board-dependent | Mmap weights |

Sources: [Espressif ESP32-P4](https://www.espressif.com/en/products/socs/esp32-p4), [ESP-DL operator support](https://raw.githubusercontent.com/espressif/esp-dl/master/operator_support_state.md), [esp-nn README](https://raw.githubusercontent.com/espressif/esp-nn/master/README.md).

**Key Takeaways**:

- One `ESP32-S3` tops out near **14 MB of weights** (flash mmap) or **6 MB** (PSRAM after KV and heap): dense int8 near 13M params, int4 near 25M.
- Every 100M+ general-purpose model is out of reach; `ESP32-P4` adds roughly 4x memory and 2..3x compute, still short of 135M at int4.
- The Waveshare `ESP32-S3FH4R2` (2 MB quad PSRAM, 2 MB SPIFFS) caps v0 fp32 checkpoints near 1.2 MB.

---

## Pretrained Models

`Q8` is `llama2.c` `Q8_0` (1.06 B/param), `Q4` is 0.56 B/param; sizes `est.` unless cited.

| Model | Params | Vocab | Q8 | Q4 | llama2.c | License | S3 Fit |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| [stories260K](https://huggingface.co/karpathy/tinyllamas/tree/main/stories260K) | 0.26M | 512 | 0.3 MB | 0.15 MB | Native | MIT | Yes (current) |
| [TinyStories-656K](https://huggingface.co/raincandy-u/TinyStories-656K) | 0.66M | 2048 | 0.7 MB | 0.4 MB | `--hf` | Apache-2.0 | Yes |
| [stories15M](https://huggingface.co/karpathy/tinyllamas) | 15.2M | 32000 | 16.2 MB | 7.5 MB | Native | MIT | Q4 only |
| [stories42M](https://huggingface.co/karpathy/tinyllamas) | 42M | 32000 | 45 MB | 24 MB | Native | MIT | No |
| [TinyStories-1M](https://huggingface.co/roneneldan/TinyStories-1M) | 3.7M | 50257 | 3.9 MB | 2 MB | No (GPT-Neo) | None tagged | Vocab prune |
| [Pythia-14m](https://huggingface.co/EleutherAI/pythia-14m) | 14M | 50304 | 15 MB | 8 MB | No (NeoX) | Apache-2.0 (`unverified`) | Poor |
| [SmolLM2-135M](https://huggingface.co/HuggingFaceTB/SmolLM2-135M) | 135M | 49152 | 143 MB | 76 MB | Patch | Apache-2.0 | No |
| [MobileLLM-R1-140M](https://huggingface.co/facebook/MobileLLM-R1-140M) | 140M | 128k | 149 MB | 79 MB | Patch | FAIR NC | No |
| [Gemma 3 270M](https://huggingface.co/google/gemma-3-270m) | 268M | 262144 | 285 MB | 151 MB | No | Gemma terms | No |
| [Qwen3-0.6B](https://huggingface.co/Qwen/Qwen3-0.6B) | 0.6B | 151936 | 640 MB | 340 MB | No (QK-norm) | Apache-2.0 | No |
| [BitNet b1.58 2B4T](https://huggingface.co/microsoft/bitnet-b1.58-2B-4T) | 2B | 128256 | n/a | 0.4 GB ternary | No | MIT | No |

**Key Takeaways**:

- **Vocabulary Overhead**: `Gemma 3 270M` is 63% embeddings (168M), and `stories15M` is 61% (`32000 x 288 = 9.2M`).
- **Usable S3 Weights**: Stay with TinyStories-class (`stories260K`, `TinyStories-656K`, `stories15M` at Q4) or self-trained domain models with compact `256..1024` SentencePiece vocabularies from [`tinyllm-training/`](tinyllm-training/README.md).
- **Export Notes**: [`export.py`](third_party/llama2.c/export.py) `--hf` loads any `LlamaForCausalLM` (GQA, tied embeddings, `Q8_0`); [`run.c`](third_party/llama2.c/run.c) hardcodes `rope_theta=10000`; [`tokenizer.py`](third_party/llama2.c/tokenizer.py) exports SentencePiece only.

---

## Community MCU Builds

| Project | Params | Format | Memory | tok/s | Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| [NanoMind-S3](https://github.com/imFARSI/NanoMind-S3) | 15.2M | int4 | 7.49 MB | 2.96 | `stories15M`, dual core |
| [slvDev/esp32-ai](https://github.com/slvDev/esp32-ai) | 28.9M | 4-bit | 14.9 MB | 9.88 | PLE in flash |
| [cardputer-ai](https://github.com/therezor/cardputer-ai) | 8M | Q4_0 | 5.7 MB flash | 5 | 13K vocab, no PSRAM |
| [pocket-tank](https://github.com/mediacutlet/pocket-tank) | 14.3M | 4-bit | 7.56 MB flash | n/a | 54-token vocab |
| [needle-2-esp32](https://github.com/andrisgauracs/needle-2-esp32) | 45M | `.cact` | 13.1 MB flash | 1.87 | Constrained tools |
| [esp32-offline-translation](https://github.com/Charles2524/esp32-offline-translation) | 3.4M | Q8_0 | 3.8 MB PSRAM | 2.5..3.3 | 4K vocab |
| [ESP32-Tiny-LLM](https://github.com/marcelpetrick/ESP32-Tiny-LLM) | 0.79M | int8, Q4 | 804, 446 KiB | n/a | GPLv3 |
| [atome-lm](https://github.com/TilelliLab/atome-lm) | 0.94M | ternary | 271 KB | 1 | ESP32-WROOM-32 |

**Key Takeaways**:

- Dense `stories15M` runs around 3 tok/s at int4; 10 tok/s at 29M needed flash-resident Per-Layer Embeddings (PLE).
- Useful MCU models are narrow: `54..4K` token vocabularies, teacher distillation, and constrained output.

---

## Quantization, Runtimes

| Scheme | Bits | 15M Size | Runtime | S3 Path |
| :--- | :--- | :--- | :--- | :--- |
| fp32 | 32 | 61 MB | `run.c` | FPU, no SIMD |
| Q8_0 (W8A8) | 8.5 | 16.2 MB | `runq.c` | PIE int8 MAC |
| Q4 (W4A8) | 4.5 | 7.5 MB | NanoMind | Unpack to int8 |
| Ternary | 1.6..2 | 3..4 MB | atome-lm | Add, subtract |

**Key Takeaways**:

- **Int8 Sweet Spot**: `ESP32-S3` PIE accelerates `int8` and `int16`, not `fp32`. `Q8_0` keeps `RMSNorm` `fp32` and quantizes activations dynamically.
- **Runtime Choice**: No vendor runtime (`ESP-DL`, `esp-nn`, `esp-tflite-micro`, `ExecuTorch`) provides KV-cached autoregressive decoding on `ESP32-S3`. Every working `S3` LLM is hand-written C (`llama2.c` / `esp32-llm`), using `ESP-DL` and `esp-nn` as SIMD kernel donors.

---

## Speedup Techniques

| Technique | Gain | Evidence |
| :--- | :--- | :--- |
| Q8 with PIE int8 | 2..4x matmul `est.` | esp-nn, ESP-DL |
| Smaller vocab | 2.5x on stories15M at 4K | Translation build |
| Flash mmap weights | Capacity up to 14 MB | needle, pocket-tank |
| SRAM activations | Moderate latency win | pocket-tank, slvDev |
| Int8 KV cache | 4x smaller KV `est.` | `2 x layers x seq x kv_dim` |
| Dual-core split | Up to 1.8x | NanoMind, cardputer |
| PLE in flash | 29M at 10 tok/s | slvDev |

**Key Takeaways**:

- **Biggest Wins**: `int8` PIE matmul, vocabulary reduction (`512..1024` tokens), and fewer cross-core syncs.
- **KV Sizing**: `stories260K` `fp32` KV cache is `655 KB` at `seq_len=512`; cutting `seq_len` to `128` drops KV to `164 KB` (`fp32`) or `41 KB` (`int8`).

---

## SAM Voice Research

[`tiny_llama_voice.ipynb`](tiny_llama_voice.ipynb) and [`third_party/SAM`](third_party/SAM/README.md) (`jake-g/SAM`, `5f54f9f`) established the original TinyLLaMA plus 1982 Don't Ask Software `SAM` voice pipeline that consumer firmware can extend onto `ESP32-S3` hardware:

- **Colab Notebook Archetypes**: The notebook defines the 6 classic 1982 manual presets (`Elf`, `Little Robot`, `Stuffy Guy`, `Little Old Lady`, `Extra-Terrestrial`, `SAM`) plus `robo-jake` (`speed=126, pitch=212, throat=116, mouth=80`) and the Colab slider profile (`speed=96, pitch=88, throat=116, mouth=124`).
- **Direct Phonetic Singing Mode (`-phonetic -sing`)**: Cell 10 (`oh_say_can_you_c()`) and [`third_party/SAM/sing`](third_party/SAM/sing) bypass `TextToPhonemes()` to feed raw SAM ARPABET codes with vowel elongation and per-syllable `-pitch` control.
- **Reciter Rule Diagnostics (`sam -debug`)**: Compound words trigger legacy 1982 `ReciterTabs.h` vowel rules (`ONLINE` to `AANLIHN`, `ZERO` to `ZEHROW`); respellings in `PHONETIC_REPLACEMENTS` (`on line` to `AAN LAY5N`, `zeero` to `ZIY4ROW`) raise ASR word recognition.
- **Clause Limit**: `SAM` input clauses stay `<= 75` characters (8-bit reciter index), which bounds generated sentence length for any on-device task.

### Project Links

#### ESP-32 Code
* my fork of [esp32-llm](https://github.com/jake-g/esp32-llm)
  * [esp32-llm](https://github.com/DaveBben/esp32-llm) original by Dave Bennet
  * [esp32-llm](https://github.com/mc9625/esp32-llm/) fork by mc9625
  * [esp32-llm](https://github.com/AIWintermuteAI/esp32-llm) fork by AIWintermuteAI
  

#### C Code
* [Tiny LLama Voice in C](https://colab.research.google.com/github/jake-g/micro-llm/blob/main/tiny_llama_voice.ipynb): my `llama.c` and `sam.c` experimental colab
  * [github source](https://github.com/jake-g/micro-llm/blob/main/tiny_llama_voice.ipynb)
  * [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/jake-g/micro-llm/blob/main/tiny_llama_voice.ipynb)
* [llama2.c](https://github.com/karpathy/llama2.c) by Andrej Karpathy
  * [huggingface dataset](https://huggingface.co/karpathy/tinyllamas/discussions)
* my fork of [sam.c](https://github.com/jake-g/SAM-colab)
  * aka *Software Automatic Mouth* from 1982, see [notes](##-Notes:) below
  * [SAM](https://github.com/vidarh/SAM) original by vidarh ([web demo](https://discordier.github.io/sam/))
  
#### Papers
* [NotebokeLM](https://notebooklm.google.com/notebook/06e729c9-b18f-4177-9e3b-858fa55d4775)
  
* LLMs
  * [TinyStories: How Small Can Language Models Be and Still Speak Coherent English?](https://arxiv.org/abs/2305.07759) (5-12-2023)
    * [huggingface dataset](https://huggingface.co/datasets/roneneldan/TinyStories)
  * [The Era of 1-bit LLMs: All Large Language Models are in 1.58 Bits](https://arxiv.org/abs/2402.17764) (3-27-2024)
  
* Vocal Track Speech Synthesis
  * Julius O. Smith III, "Physical audio signal processing for virtual musical instruments and audio effects." https://ccrma.stanford.edu/~jos/pasp/
  * Story, Brad H. "A parametric model of the vocal tract area function for vowel and consonant simulation." The Journal of the Acoustical Society of America 117.5 (2005): 3231-3254.
  * Lu, Hui-Ling, and J. O. Smith. "Glottal source modeling for singing voice synthesis." Proceedings of the 2000 International Computer Music Conference. 2000.
  * Mullen, Jack. Physical modelling of the vocal tract with the 2D digital waveguide mesh. PhD thesis, University of York, 2006.

#### Datasheets
* Waveshare ESP32s3 [ESP32-S3FH4R2](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf) ([amazon](https://www.amazon.com/gp/product/B0CHYHGYRH?th=1))
* Speech Synthesis
  * [Votrax SC-01 Data Sheet](http://www.bitsavers.org/pdf/federalScrewWorks/Votrax_SC-01_Phoneme_Speech_Synthesizer_Data_Sheet_1980.pdf) (1980)
  * [SAM](http://www.apple-iigs.info/newdoc/sam.pdf) (1982)
  * [SP0256A-AL2 datasheet](bitsavers.org/components/gi/speech/General_Instrument_-_SP0256A-AL2_datasheet_(Radio_Shack_276-1784)_-_Apr1984.pdf) (1984)
  * [SpeaknSpell 80](http://www.datamath.org/Speech/SpeaknSpell_80.htm) (1980)
## Notes:

### Speech synth unsorted
* imaginary.org/program/pink-trombone
  * https://github.com/IMAGINARY/pink-trombone
* ESP speech synth
	* https://lucstechblog.blogspot.com/2022/11/talkie-part-1-esp32-speech-synthesiser.html
	https://github.com/bobh/ESP32Talkie
	* https://cdn-learn.adafruit.com/downloads/pdf/bringing-back-the-voice-of-speak-spell.pdf

### llama2.c models

From [README.md](https://github.com/karpathy/llama2.c/blob/master/README.md)

For the sake of examples of smaller, from-scratch models, I trained a small model series on TinyStories. All of these trained in a few hours on my training setup (4X A100 40GB GPUs). The 110M took around 24 hours. I am hosting them on huggingface hub [tinyllamas](https://huggingface.co/karpathy/tinyllamas), both in the original PyTorch .pt, and also in the llama2.c format .bin:

| model | dim | n_layers | n_heads | n_kv_heads | max context length | parameters | val loss | download
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 260K | 64 | 5 | 8 | 4 | 512 | 260K | 1.297 | [stories260K](https://huggingface.co/karpathy/tinyllamas/tree/main/stories260K)
| OG | 288 | 6 | 6 | 6 | 256 | 15M | 1.072 | [stories15M.bin](https://huggingface.co/karpathy/tinyllamas/resolve/main/stories15M.bin) |
| 42M| 512 | 8 | 8 | 8 | 1024 | 42M | 0.847 | [stories42M.bin](https://huggingface.co/karpathy/tinyllamas/resolve/main/stories42M.bin) |
| 110M| 768 | 12 | 12 | 12 | 1024 | 110M | 0.760 | [stories110M.bin](https://huggingface.co/karpathy/tinyllamas/resolve/main/stories110M.bin) |

You'll notice that the 110M model is equivalent to GPT-1 in size. Alternatively, this is also the smallest model in the GPT-2 series (`GPT-2 small`), except the max context length is only 1024 instead of 2048. The only notable changes from GPT-1/2 architecture is that Llama uses RoPE relatively positional embeddings instead of absolute/learned positional embeddings, a bit more fancy SwiGLU non-linearity in the MLP, RMSNorm instead of LayerNorm, bias=False on all Linear layers, and is optionally multiquery.

### Classic Speech Synthesis Overview

More details: [speech_synthesis_algorithms](https://pichenettes.github.io/mutable-instruments-documentation/tech_notes/speech_synthesis_algorithms/)

**Votrax SC-01:**
* **Date:** ~1978
* **Technology:** Phoneme synthesis
* **Sound:** Robotic, metallic
* **Products:** MicroVox, Heathkit HERO 1
* **Note:**  Votrax was originally Federal Screw Works, and their transition to speech synthesis was driven by discoveries related to an electric shaver project.
* **Datasheet:** [Votrax SC-01 Datasheet](http://www.bitsavers.org/pdf/federalScrewWorks/Votrax_SC-01_Phoneme_Speech_Synthesizer_Data_Sheet_1980.pdf)


**Texas Instruments TMS5100 (Speak & Spell aka Talkie):**
* **Date:** ~1978
* **Technology:** Linear Predictive Coding (LPC)
* **Sound:** Clear, natural, childlike
* **Products:** Speak & Spell
* **Patent:** US4209844A  (related to LPC speech analysis/synthesis)
* **Paper:**  Related research published earlier, for example:  "Linear prediction of speech: A tutorial," by J. Markel and A. Gray, Jr. (IEEE ASSP Magazine, 1976)
* **Datasheet:** [SpeaknSpell 80](http://www.datamath.org/Speech/SpeaknSpell_80.htm)
  
**General Instrument SPO256-AL2:**
* **Date:** ~1980
* **Technology:** Allophone synthesis
* **Sound:** Robotic, smoother than Votrax
* **Products:** Speak & Math, various consumer products, sold by Radio Shack
* **Patent:** US4196351A
* **Datasheet:**  [SP0256A-AL2 datasheet](bitsavers.org/components/gi/speech/General_Instrument_-_SP0256A-AL2_datasheet_(Radio_Shack_276-1784)_-_Apr1984.pdf)

**S.A.M. (Software Automatic Mouth):**
* **Date:** ~1982
* **Technology:** Software-based formant synthesis
* **Sound:** Buzzy, nasal, more expressive than Votrax
* **Products:** Software for Apple II computers
* **Note:** Developed by Don't Ask Software (later Softalk).
* **Datasheet:** [manual](https://github.com/discordier/sam/blob/master/docs/manual.md) [pdf](http://www.apple-iigs.info/newdoc/sam.pdf)
### SAM: Software Automatic Mouth - Tiny Speech Synthesizer

Taken from original [README.md](https://github.com/vidarh/SAM/blob/master/README.md), also see [wikipedia](https://en.wikipedia.org/wiki/Software_Automatic_Mouth)


#### What is SAM?

Sam is a very small Text-To-Speech (TTS) program written in C, that runs on most popular platforms.
It is an adaption to C of the speech software SAM (Software Automatic Mouth) for the Commodore C64 published 
in the year 1982 by Don't Ask Software (now SoftVoice, Inc.). It includes a Text-To-Phoneme converter called reciter and a Phoneme-To-Speech routine for the 
final output. It is so small that it will work also on embedded computers. On my computer it takes
less than 39KB (much smaller on embedded devices as the executable-overhead is not necessary) of disk space and is a fully stand alone program. 
For immediate output it uses the SDL-library, otherwise it can save .wav files. 

An online version and executables for Windows can be found on the web site: http://simulationcorner.net/index.php?page=sam

#### Compile

Simply type "make" in your command prompt.
In order to compile without SDL remove the SDL statements from the CFLAGS and LFLAGS variables in the file "Makefile".

It should compile on every UNIX-like operating system. For Windows you need Cygwin or MinGW( + libsdl).


#### Usage

type

	./sam I am Sam

for the first output.

If you have disabled SDL try

	./sam -wav i_am_sam.wav I am Sam

to get a wav file. This file can be played by many media players available for the PC.

you can try other options like
	-pitch number
	-speed number
	-throat number
	-mouth number

Some typical values written in the original manual are:

	DESCRIPTION          SPEED     PITCH     THROAT    MOUTH
	Elf                   72        64        110       160
	Little Robot          92        60        190       190
	Stuffy Guy            82        72        110       105
	Little Old Lady       82        32        145       145
	Extra-Terrestrial    100        64        150       200
	SAM                   72        64        128       128


It can even sing
look at the file "sing"
for a small example.

For the phoneme input table look in the Wiki.


A description of additional features can be found in the original manual at
	http://www.retrobits.net/atari/sam.shtml
or in the manual of the equivalent Apple II program
	http://www.apple-iigs.info/newdoc/sam.pdf

