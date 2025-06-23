# Micro LLMs [wip]

by [jake-g](https://github.com/jake-g)

Project root for working with various forks and experiemnts for Micro LLM project. See `.gitmodules` for included repositories.

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

