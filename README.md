# tesi-hfref
Codice finalizzato alle fasi di estrazione delle informazioni da lettere cliniche e calcolo degli score di aderenza per l'HFrEF. Tesi di laurea magistrale in Bioingegneria: "Tecniche di intelligenza artificiale per l’estrazione strutturata di informazioni da testi clinici per il supporto all’aderenza delle prescrizioni terapeutiche"

Il progetto si compone di tre notebook, da eseguire in quest'ordine:

1. **`TESI_multimodal_extraction.ipynb`**: estrae da un'immagine (tabella delle linee guida) le dosi iniziali e target dei farmaci per l'HFrEF.
2. **`TESI_IE.ipynb`**: estrae informazioni strutturate dalle lettere cliniche con modelli linguistici (Ollama o Amazon Bedrock).
3. **`TESI_GDMT_scores.ipynb`**: confronta la terapia estratta con le dosi target e calcola gli score di aderenza alla terapia.

Le cartelle `HFrEF letters/` e `HFrEF_results/` **non sono incluse**:
vanno create a mano nella cartella del progetto.

## Requisiti e configurazione

- Python 3.14.5 o successivo.

- Variabili d'ambiente: copia `.env.example` in un file chiamato `.env` (nella stessa cartella dei notebook) e compila i valori.

---

## `TESI_multimodal_extraction.ipynb`: dosi dalle linee guida

**Cosa fa.** Legge un'immagine con la tabella delle dosi dei farmaci per l'HFrEF e la trasforma in dati strutturati,
usando un modello multimodale su Amazon Bedrock (`qwen.qwen3-vl-235b-a22b`). L'output strutturato è ottenuto
con la funzione di *tool use* di Bedrock, che obbliga il modello a rispondere secondo uno schema `pydantic`.

**Passaggi.**
1. **Elenco dei farmaci**: per ogni classe (`ACE-I`, `ARNI`, `Beta-blockers`, `MRA`, `SGLT2 inhibitor`, `Other agents`) estrae i nomi generici.
2. **Tabella completa**: per ogni farmaco estrae dose iniziale, dose target e relativo numero di somministrazioni giornaliere.

**Input.**
- `images/HFrEF-medications-table.png`
- `prompts/HFrEF_medication_list_prompt.txt`

**Output** (nella cartella `data/`, che deve esistere prima di eseguire il notebook):
- `HFrEF_medication_list.json` e `.csv`
- `HFrEF_medication_table.json` e `.csv`

I file prodotti sono già presenti nel repository.

---

## `TESI_IE.ipynb`: estrazione di informazioni dalle lettere cliniche

**Cosa fa.** Trasforma il testo di una lettera clinica in un JSON che rispetta uno schema `pydantic`.
Ogni campo estratto contiene il valore (`value`) e la frase della lettera da cui proviene (`source_text`),
così ogni dato è verificabile.

**Contenuto del notebook.**
- **Schema dei dati**: terapie (attuale e alla dimissione, con farmaco, dose, numero di somministrazioni e classe), valori di laboratorio,
  comorbidità e altre informazioni cliniche. Il notebook contiene prima lo schema base ("Original variables")
  e poi la versione con il testo di origine ("with text span"), che è quella usata dalla pipeline.
- **Lettura delle lettere** in formato `.docx`, `.txt` e `.rtf`.
- **Prompt di sistema** (`system_prompt.txt`): contiene il segnaposto `$output_schema`, sostituito con lo schema JSON.
- **Due backend**:
  - *Local LLM (Ollama)*: modelli testati `gpt-oss:120b-cloud`, `gemma4:31b-cloud`.
  - *AWS LLM (Amazon Bedrock)*: modelli `deepseek.v3.2`, `qwen.qwen3-32b-v1:0`, `us.anthropic.claude-sonnet-4-6`.
- **Pulizia della risposta**: rimozione dei caratteri di controllo e dei blocchi markdown prima di leggere il JSON.

**Come si usa.**
1. Crea la cartella `HFrEF letters/` e inserisci le lettere in formato `.docx`.
2. Nella lista `HFrEF_Letter_1` scrivi i nomi dei file **senza estensione** e scegli i modelli in `model_names`.
3. Esegui la cella del backend scelto (Ollama oppure AWS).

**Output.** Un file per ogni coppia lettera/modello in `HFrEF_results/`:
`<lettera>_<modello>_with_span.json` (i `:` nel nome del modello diventano `_`).
Un errore su una singola coppia viene stampato e non blocca le altre.

---

## `TESI_GDMT_scores.ipynb`: score di aderenza alla terapia

**Cosa fa.** Confronta la terapia estratta dalle lettere con le dosi target delle linee guida e calcola tre score:
**mHFC 2022**, **OPS** e **Modified C4P**. Usa solo la libreria standard di Python.

**Contenuto del notebook.**
- **Utility di parsing delle dosi**: gestiscono valori singoli (`6.25 mg`), range (`10–20 mg`) e combinazioni (`97/103 mg`),
  e calcolano la dose giornaliera totale (dose per unità × unità al giorno).
- **Abbinamento dei farmaci** alla tabella delle dosi target a partire dal nome generico.
- **Funzioni di calcolo** dei tre score e una funzione che li calcola insieme (`calculate_adherence_scores`).
- **Calcolo su una lettera**, per la terapia attuale (`current_therapy`) e per quella alla dimissione (`discharge_therapy`),
  con tabella riassuntiva e dettaglio farmaco per farmaco.
- **Confronto tra modelli** per uno stesso paziente: cerca in `HFrEF_results/` tutti i file
  `pz_<ID>_cardiologia_<modello>_with_span.json` e calcola gli score per ciascun modello.

**Input.**
- `data/HFrEF_medication_table.json` (dosi target, incluso nel repository)
- i JSON prodotti da `TESI_IE` in `HFrEF_results/` (non inclusi)

**Parametri da modificare** nelle celle di esecuzione: `CLINICAL_LETTER_PATH`, `PATIENT_ID`, `MODEL_NAME`.
