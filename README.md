# 👩‍🍳 AI Sous Chef — Generative AI Kitchen Assistant

A conversational kitchen assistant that answers *"What should I cook with what I have?"*. Tell it your ingredients, dislikes, diet or the cuisine you're in the mood for, in plain English, and it suggests a recipe.

Built as my capstone for the **Google × Kaggle 5-Day Generative AI Intensive**.

## How it works

```
User message
   │
   ▼
Gemini (few-shot + JSON schema) ──► structured preferences
   │                                 {ingredients, dislikes, diet, cuisine, difficulty, dish}
   │
   ├─ ingredients given ──► filter RecipeNLG (pandas)
   │                         └─► embed top 100 (Gemini embeddings) ──► ChromaDB
   │                               └─► vector search ──► best recipe + "you may be missing…"
   │
   └─ only high-level prefs / no local match ──► Gemini + Google Search grounding
                                                  └─► dish suggestions ──► grounded recipe
```

**GenAI techniques used:** structured output / JSON mode · few-shot prompting · embeddings · vector search (ChromaDB) · RAG · Google Search grounding.

| Component | Tool |
|---|---|
| LLM | `gemini-2.0-flash` via `google-genai` |
| Embeddings | `text-embedding-004` |
| Vector store | ChromaDB (in-memory) |
| Data | [RecipeNLG](https://recipenlg.cs.put.poznan.pl/) (~2.2M recipes) |

## Repo structure

```
kitchen-assistant-genAI/
├── ai-sous-chef-generative-ai.ipynb   # the full project
├── README.md
└── .gitignore                         # excludes datasets + API keys
```

> Datasets are **not** included in this repo (the main CSV is ~2.2 GB). Follow the steps below to download it.

---

## Getting started

You can run the notebook on **Kaggle** (easiest, and how it was built) or **locally**.

### 1. Get a Gemini API key

1. Go to [Google AI Studio → API keys](https://aistudio.google.com/apikey).
2. Sign in and click **Create API key**.
3. Copy the key. Treat it like a password and never paste it into the notebook or commit it.

### 2. Get the dataset

The notebook uses `full_dataset.csv` from **RecipeNLG**.

1. Go to <https://recipenlg.cs.put.poznan.pl/dataset>.
2. Accept the terms and download the zip.
3. Unzip it to get `full_dataset.csv` (~2.2 GB).

(Dataset card: [mbien/recipe_nlg on Hugging Face](https://huggingface.co/datasets/mbien/recipe_nlg).)

---

### Option A: Run on Kaggle (no code changes)

1. Create a new Kaggle notebook and import `ai-sous-chef-generative-ai.ipynb` (**File → Import Notebook**).
2. **Add the dataset:** upload `full_dataset.csv` as a Kaggle dataset named **`recipes`** and attach it to the notebook, so the path is `/kaggle/input/recipes/full_dataset.csv`.
3. **Add your API key:** **Add-ons → Secrets → Add a new secret**, with label `GOOGLE_API_KEY` and your key as the value. Tick the checkbox to attach it to the notebook.
4. Turn **Internet on** in the notebook settings (needed for the Gemini API and Google Search grounding).
5. **Run All**, then chat in the input box at the bottom. Type `exit` to stop.

### Option B: Run locally

**Requirements:** Python 3.10+, Jupyter.

```bash
git clone https://github.com/salehak/kitchen-assistant-genAI.git
cd kitchen-assistant-genAI

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install "google-genai==1.7.0" "chromadb==0.6.3" pandas python-dotenv jupyter
```

**Put the dataset in a `data/` folder** (it's git-ignored):

```
kitchen-assistant-genAI/
└── data/
    └── full_dataset.csv
```

**Store your API key in a `.env` file.** See [Storing your API key locally](#-storing-your-api-key-locally) below.

**Make two small edits in the notebook.** The notebook is written for Kaggle, so swap these two cells when running locally:

| Cell | Kaggle version (current) | Local version |
|---|---|---|
| **API Setup** | `from kaggle_secrets import UserSecretsClient`<br>`GOOGLE_API_KEY = UserSecretsClient().get_secret("GOOGLE_API_KEY")` | `import os`<br>`from dotenv import load_dotenv`<br>`load_dotenv()`<br>`GOOGLE_API_KEY = os.environ["GOOGLE_API_KEY"]` |
| **Load and Filter Recipes** | `pd.read_csv('/kaggle/input/recipes/full_dataset.csv')` | `pd.read_csv('data/full_dataset.csv')` |

You can also skip the first `!pip uninstall …` cell; it only removes Kaggle-specific packages.

Then:

```bash
jupyter notebook ai-sous-chef-generative-ai.ipynb
```

> ⚠️ Loading the full 2.2 GB CSV needs a fair amount of RAM (roughly 8 GB+). If it struggles, test with a sample, e.g. `pd.read_csv('data/full_dataset.csv', nrows=200_000)`.

---

## 🔐 Storing your API key locally

**Recommended: a `.env` file in the project root.**

1. Create a file called `.env` next to the notebook:

   ```bash
   echo 'GOOGLE_API_KEY=your-key-here' > .env
   ```

   Contents:

   ```
   GOOGLE_API_KEY=your-key-here
   ```

   (No quotes, no spaces around `=`.)

2. `.env` is already in `.gitignore`, so it will not be committed. Check this before your first push:

   ```bash
   git status            # .env should NOT appear
   git check-ignore .env # should print ".env"
   ```

3. In code, `load_dotenv()` reads it and `os.environ["GOOGLE_API_KEY"]` returns the key.

**Alternative: a shell environment variable** (available to every project, no file in the repo):

```bash
# macOS (zsh): add to ~/.zshrc, then restart the terminal or run `source ~/.zshrc`
export GOOGLE_API_KEY="your-key-here"
```

`os.environ["GOOGLE_API_KEY"]` works the same way. `load_dotenv()` does nothing if there's no `.env`.

**If a key ever gets committed:** delete it in AI Studio and create a new one. Removing it in a later commit is not enough because it stays in git history.

---

## Example prompts

- *"I have tomatoes, spinach and eggs, what can I cook?"*
- *"Make me something vegetarian without olives. Something Italian maybe?"*
- *"I want to bake something sweet"*
- *"I have chicken and rice, give me something easy"*

## Roadmap

- [ ] **Agentic chat instead of a scripted flow.** Today the conversation follows fixed branches with hard-coded follow-up prompts (`"Which one would you like to make?"`, `"Are you happy with this recipe? (yes/no)"`). The plan is to turn the existing functions (`extract_preferences`, `filter_and_embed_recipes`, `find_best_recipe`, `grounded_google_search`, …) into **tools** and let a Gemini agent with function calling decide what to call and what to ask next, keeping the conversation history.
- [ ] **Ingredient expansion / similar ingredients.** Dislikes are currently matched as literal substrings, so "no mushrooms" doesn't reliably exclude *shiitake*, *portobello* or *cremini*. The plan is to expand each ingredient into its family (via an LLM call, a curated synonym map, or embedding similarity over the `NER` ingredient vocabulary) before filtering, for both dislikes and available ingredients.
- [ ] Pre-compute and persist recipe embeddings (offline ChromaDB index) so recipes aren't re-embedded per request.
- [ ] Move from notebook to a deployable app (FastAPI backend + web UI).

## Acknowledgements

- [Google × Kaggle 5-Day Gen AI Intensive](https://www.kaggle.com/learn-guide/5-day-genai): embedding-function code adapted from the course codelabs.
- Bień et al., *RecipeNLG: A Cooking Recipes Dataset for Semi-Structured Text Generation* (INLG 2020).
