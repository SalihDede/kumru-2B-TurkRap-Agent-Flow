# Fincan Kahvem Hatrina

**Fully autonomous AI rap song generation pipeline** — Two fine-tuned LoRA personas (Sagopa Kajmer & Ceza) write, critique, and revise lyrics together through a multi-agent LangGraph workflow.

> Kumru-2B Turkish LLM + LoRA adapters generate verses, Claude Haiku 4.5 orchestrates ideation/critique/hooks, and a shared memory system enables real-time cross-artist awareness during parallel writing.

## Architecture

```
Claude Haiku 4.5 (Orchestrator)
  ├─► IDEATION ─── theme, mood, directives
  ├─► STYLE PROFILER ─── artist-specific style prompts
  └─► PRODUCER ─── song structure & section briefs
          │
      LangGraph
          │
   ┌──────┴──────┐
   ▼              ▼
 Kumru-2B       Kumru-2B
 Sagopa LoRA ◄──► Ceza LoRA      ← SharedMemory (thread-safe)
   └──────┬──────┘
          ▼
    VERSE CRITIC (Claude Haiku)
     │ fail → both rewrite ↑
     │ pass ↓
    HOOK WRITER (Claude Haiku)
          ▼
    HOOK CRITIC (Claude Haiku)
     │ fail → hook rewrites ↑
     │ pass ↓
    ASSEMBLY (Claude Haiku)
          ▼
    BEAT SUGGESTER (Claude Haiku)
          ▼
         END
```

### Key Design Decisions

- **Parallel fan-out with shared memory**: Sagopa and Ceza write simultaneously via LangGraph parallel nodes. A thread-safe `SharedMemory` class lets each artist read the other's verses on-the-fly, enabling real-time stylistic coordination outside LangGraph state.
- **All-or-nothing revision**: If *any* verse fails the critic threshold, *both* artists rewrite — preserving cross-artist harmony. Passing verses are kept in shared memory as context for the next iteration.
- **Hook after verses**: The hook (nakarat) is only written after all verses are approved, ensuring it reflects the final lyrical content.
- **Hook critic loop**: The hook gets its own evaluate-rewrite cycle against the full song context.
- **Score normalization**: `safe_score()` handles LLM output quirks (e.g., `72` → `7.2`) to prevent infinite revision loops.

## Models

| | Sagopa Kajmer | Ceza |
|---|---|---|
| **Base Model** | [vngrs-ai/Kumru-2B](https://huggingface.co/vngrs-ai/Kumru-2B) | [vngrs-ai/Kumru-2B](https://huggingface.co/vngrs-ai/Kumru-2B) |
| **LoRA Rank** | 16 | 8 |
| **LoRA Alpha** | 32 | 16 |
| **Dropout** | 0.05 | 0.3 |
| **Target Modules** | q/k/v/o_proj, gate/up/down_proj | q/k/v/o_proj, gate/up/down_proj |
| **Style** | Melankolik, meditativ, felsefi metaforlar | Hizli flow, sert punchline, agresif enerji |

**Weights:** [SalihHub/Kumru_2B_TurkRap](https://huggingface.co/SalihHub/Kumru_2B_TurkRap)

### Training

- **Method:** SFT with PEFT/LoRA
- **Data:** ~120 Sagopa Kajmer lyrics + ~100 Ceza lyrics, manually curated
- **Format:** ChatML (`<|im_start|>user` / `<|im_end|>`)
- **Framework:** Transformers + PEFT 0.18.1
- **Hardware:** Single T4 GPU (Google Colab)

## Quick Start (Colab)

1. Open `Kumru_Colab_Pipeline.ipynb` in Google Colab (T4 GPU runtime)
2. Create a `.env` file with your OpenRouter API key (see `example.env`)
3. Upload LoRA weights from [SalihHub/Kumru_2B_TurkRap](https://huggingface.co/SalihHub/Kumru_2B_TurkRap) or let the notebook pull from Google Drive
4. Run all cells — the pipeline will autonomously generate a full song

## Project Structure

```
kumru-2B-TurkRap-Agent-Flow/
├── Kumru_Colab_Pipeline.ipynb  # Full pipeline notebook (Colab-ready)
├── example.env                 # Environment template
├── .gitignore
└── README.md
```

## Pipeline Flow

1. **Ideation** — Claude Haiku generates an original theme, mood, and creative directives
2. **Style Profiler** — Creates artist-specific style prompts for each persona
3. **Producer** — Plans song structure (intro, verses, hooks, outro) with section briefs
4. **Parallel Writing** — Sagopa & Ceza LoRA models write verses simultaneously, reading each other's output via shared memory
5. **Verse Critic** — Claude Haiku evaluates each verse for style authenticity, cross-artist harmony, and thematic coherence
6. **Revision Loop** — If any verse scores below threshold (6.5/10), both artists rewrite with critic feedback
7. **Hook Writing** — Once verses pass, Claude Haiku writes a hook that bridges both artists' styles
8. **Hook Critic** — Evaluates the hook against the full song context, rewrites if needed
9. **Assembly** — Combines all approved sections into the final song
10. **Beat Suggestion** — Recommends BPM, key, instruments, and production style

## Tech Stack

| Component | Technology |
|---|---|
| Base LLM | [Kumru-2B](https://huggingface.co/vngrs-ai/Kumru-2B) (Turkish, 2B params) |
| Fine-tuning | PEFT/LoRA |
| Orchestrator | Claude Haiku 4.5 via OpenRouter |
| Workflow | LangGraph (StateGraph with conditional edges) |
| Cross-node sync | Thread-safe SharedMemory |
| UI | Streamlit |
| Deployment | Google Colab / RunPod Serverless |

## Example Output

**Theme:** *Iki kisinin zaman algisinin farkliligi*

**Sagopa Kajmer:**
> Abi beat'ler donerken studyo los isikta, ritim beni vuruyor gece yarisi geceye dogru. Zamani unuttum burada, hizli akan bir nehrin ustunde surukleniyoruz sanki...

**Ceza:**
> Aslinda, bu flow yavas akan nehir gibi – bir yanda ruzgarin nefesi, obur yanda dalgalarin ofkesi var ya... Zaman ayni akmiyor be! Sen zamanin kolesisin, bense ritmin kaptani...

**Hook:**
> Saat durdu, nehir akti, ritim bizi tasidi / Sen saatlara bak, ben vuruslara bastim / Zaman bir degil bize, iki renk bir tuval

## License

Apache 2.0
