# ATS-Proof CV

[Leia este guia em português](README.pt-BR.md)

ATS-Proof CV tailors résumés with **local vector search over your career history**, followed by evidence scoring and a separate review before PDF export.

## Vector search and evidence selection

Each job requirement becomes a query. The skill generates embeddings with `text-embedding-3-large`, stores the career vectors in SQLite, and computes cosine similarity locally. A lexical search weighted by term rarity complements the vector ranking. The combined results form the candidate set for scoring.

`gpt-6-luna` scores each retrieved account against the requirements on a 1–5 scale. The selector groups accounts of the same accomplishment within a role, prioritizes the strongest evidence for each requirement, and uses maximal marginal relevance (MMR) to choose complementary accounts. It rewards new facts and useful quantitative details and penalizes redundancy. A compound requirement can draw on several sources. The selection review records direct, partial, transferable, and missing evidence. Any changes must preserve the highest eligible score already covered for each requirement.

## Review before PDF export

A separate model call checks whether the summary and bullet order reflect the role’s main responsibilities. The workflow checks that the figures from the selected sources are preserved and applies corrections only to the passages flagged for revision. It records added context or responsibilities for human factual review.

## API cost

**Running the skill requires paid use of the OpenAI API. The measured cost in Brazil was BRL 0.20 per résumé.** The total in Brazilian reais varies with token usage and the exchange rate. [OpenAI API pricing](https://developers.openai.com/api/docs/pricing)

## Install

1. Download or clone this repository and open the `ats-proof-cv` folder. To clone from the terminal, run `git clone https://github.com/alexsilvacw-bot/ats-proof-cv.git`.
2. Use Python 3.10 or later and LibreOffice on macOS or Linux. Make `soffice` or `libreoffice` available on your command path. On Windows, use a Linux environment such as WSL and install the dependencies there.
3. From the skill folder, run `python -m pip install -r requirements.txt` in your Python environment. If your system uses `python3`, substitute it for `python` in the examples.
4. Set `OPENAI_API_KEY` in the environment that runs the script. Your key needs access to `gpt-6.1-sol` for extraction, `gpt-6-luna` for scoring, drafting, and review, and `text-embedding-3-large` for vectors. [API authentication](https://developers.openai.com/api/reference/overview#authentication)
5. For Codex, copy the skill folder into `.agents/skills/` in your project and invoke `$ats-proof-cv`. For the terminal, run the commands below from the skill folder. [Codex skills](https://developers.openai.com/codex/skills)

## Add and index your experience

- **With Codex.** Attach your résumé or paste your accomplishments and send this request. “Use `$ats-proof-cv` to record my experience in `dados/carreira.json`. Keep the job titles, dates, and metrics I provided. Show me the career file for review before indexing.”
- **By hand.** Copy [the fictional career file](examples/carreira.json) to `dados/carreira.json` and replace the sample data with yours. The field `contato` is optional. Create one role for each employer, job title, and period. Add each account of work as a separate record in `experiencia`. Keep role and experience IDs stable and distinct.

The JSON field names and terminal flags below match the script. Your experience descriptions can be written in your own language.

| Field | What to enter |
| --- | --- |
| `nome` | Your name, as it should appear in the document header |
| `contato` | Optional contact details |
| `experiencias` | The list of roles. Each role has `id`, `empresa` (employer), `cargo` (job title), `periodo` (dates), and the `experiencia` list |
| `periodo` | Use `YYYY-MM` for `entrada` (start) and `saida` (end). Use `null` for an unknown month. For a current role, set `atual` to `true` and leave `saida` as `null` |
| `experiencia` | Each account has an `id` and `texto`. Describe what you did, the context, and the result if known. Preserve units and qualifiers for numeric claims |

Review the JSON, then prepare the career file. Keep your data and generated documents in your private workspace.

```bash
python scripts/cv.py index --base dados/carreira.json
```

Indexing sends your experience descriptions to OpenAI to group accounts of the same work and create vectors. The index and working files are stored in `.ats-proof-cv/`. Give each new experience a new ID. When editing an existing experience, keep its ID. Run `index` again after editing the file. Vectors for unchanged text with the same ID are reused, and the grouping is updated to match the current career file. To use another folder, pass the same `--workspace` path to each command.

## Create and resume an application

Save the job ad in a UTF-8 text file or pass the public job URL with `--url`.

```bash
python scripts/cv.py create --vaga dados/vaga.txt
```

The terminal prints the application folder. After an interruption, pass that same path. The workflow continues from the pending step and can reuse compatible responses from the temporary checkpoint.

```bash
python scripts/cv.py resume --candidatura .ats-proof-cv/candidaturas/company_role
```

Selection usually works from a subset of the career file. If retrieval returns every record in a small file, `--permitir-base-integral` authorizes scoring that set. In this case, the API can receive all retrieved descriptions for scoring.

### Try the fictional data

Both commands make API calls and use a separate workspace for the example.

```bash
python scripts/cv.py index --base examples/carreira.json --workspace dados/demo
python scripts/cv.py create --vaga examples/vaga.txt --workspace dados/demo --permitir-base-integral
```

## Review the result

The workflow produces a Portuguese or English PDF based on the vacancy. The DOCX is the intermediate file used for conversion. The application folder also contains these JSON records.

- `requisitos-vaga.json` contains the extracted requirements and their priorities.
- `experiencias-aprovadas.json` records the selected experiences and coverage by component. Its name refers to the selection step. Factual approval comes from the user.
- `dados-reescritos.json` contains the draft, the alignment review, and additions awaiting factual review.
- `origens-recuperacao.json` records where the retrieved experiences came from.

Open the PDF and compare it with the JSON records, paying particular attention to metrics, attribution, and pending additions. The skills and tools lists come from the job ad and also need factual review. PDF conversion and text extraction checks run locally. During an application, OpenAI receives the job description, requirements, and relevant evidence for extraction, embeddings, input token counting, scoring, and drafting. The name and contact details in the document header are filled in locally.

[Operational skill instructions](SKILL.md) · [MIT license for the public edition](LICENSE)
