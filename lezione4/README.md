# Lezione 4: Introduzione a Snakemake e Automazione delle Pipeline

## Introduzione

Nelle lezioni precedenti abbiamo eseguito molti comandi uno dopo l'altro nel terminale.
Nella vera bioinformatica non lanciamo i comandi a mano. Se abbiamo 100 campioni,
o se un passaggio fallisce a metà, o se modifichiamo un parametro iniziale, rieseguire tutto manualmente è impensabile.

La soluzione è usare un **Workflow Management System**. Oggi vedremo **[Snakemake](https://snakemake.readthedocs.io/)**, uno dei tool più usati al mondo per creare pipeline bioinformatiche.

Snakemake è basato su Python e funziona tramite regole.
Tu definisci l'**input** che hai, l'**output** che vuoi ottenere e il comando (**shell**) relativo.
Snakemake calcola automaticamente l'ordine in cui eseguire i comandi costruendo un DAG delle dipendenze.

## Ambiente

Per prima cosa, attiviamo il nostro ambiente e installiamo Snakemake (se non lo avete già fatto):

```shell
mamba activate lab
mamba install -c conda-forge -c bioconda snakemake -y
```

Creiamo una cartella pulita per questa lezione ed entriamoci:

```shell
mkdir code
cd code
```

## Le Basi di Snakemake

Un file di Snakemake si chiama tipicamente `Snakefile`. Al suo interno scriveremo delle regole. La sintassi base di una regola è questa:

```python
rule nome_della_regola:
    input:
        "file_di_partenza.txt"
    output:
        "file_risultato.txt"
    shell:
        "comando_da_eseguire {input} > {output}"
```

Snakemake lavora "a ritroso".
L'idea è dire a Snakemake il file finale, e lui cerca la regola che produce quel file. Se a quella regola mancano gli `input`, cercherà un'altra regola che produca quegli `input`, e così via.

Una regola può essere molto complessa con vari ca,pi, tra cui:

- **`input` e `output`**: I file di partenza e di arrivo. Si possono usare **nomi** a questi file (es. `bam="file.bam"`).
- **`params`**: Parametri testuali o numerici che non sono file.
- **`threads`**: Il numero di core CPU richiesti dalla regola.
- **`log`**: Il percorso dove salvare l'output a schermo (stdout) e gli errori (stderr) del tool..
- **`benchmark`**: Genera un file `.tsv` con le statistiche di esecuzione (tempo impiegato, picco di RAM utilizzata).
- **`conda`**: Permette di specificare un file `ambiente.yaml`.
- **`shell`/`script`/`run`**:
  - `shell`: Esegue comandi Bash.
  - `script`: Permette di eseguire uno script Python o R esterno (es. `script: "plot.py"`). Le variabili della regola saranno accessibili direttamente nello script tramite l'oggetto `snakemake` (es. `snakemake.input[0]`).
  - `run`: Permette di scrivere codice Python puro direttamente dentro il `Snakefile`.

## Un Primo Esempio

```python
rule all:
    input:
        "messaggio_maiuscolo.txt"

rule crea_messaggio:
    output:
        "messaggio.txt"
    shel:
        "echo 'Ciao, benvenuti alla lezione su Snakemake!' > {output}"

rule rendi_maiuscolo:
    input:
        "messaggio.txt"
    output:
        "messaggio_maiuscolo.txt"
    threads: 1
    benchmark:
        "benchmarks/rendi_maiuscolo.txt"
    conda:
        "env.yaml"
    shell:
        "tr '[:lower:]' '[:upper:]' < {input} > {output}"
```

1. Legge la prima regola (`rule all`) e vede che l'obiettivo è avere `messaggio_maiuscolo.txt`.
2. Cerca una regola che lo produca e trova `rendi_maiuscolo`.
3. Vede che `rendi_maiuscolo` ha bisogno di `messaggio.txt` come `input`.
4. Cerca una regola che produca `messaggio.txt` e trova `crea_messaggio`.
5. `crea_messaggio` non ha input richiesti, quindi Snakemake esegue prima `crea_messaggio` e poi `rendi_maiuscolo`.

### Esecuzione dell'Esempio

**Il Dry-Run**
Prima di eseguire, simuliamo il processo con `-n`. È fondamentale per controllare che il grafo delle dipendenze sia corretto.

```shell
snakemake -np --use-conda
```

Il terminale vi mostrerà in verde il piano di esecuzione, con l'ordine esatto delle regole che intende avviare.

### Esecuzione parallela (Core vs Jobs)

Lanciamo la pipeline. Esistono due flag principali per le risorse:

- `--cores N` (o `-c N`): Quanti core locali del vostro PC Snakemake può usare al massimo per parallelizzare.
- `--jobs N` (o `-j N`): Simile a `cores`, ma usato tipicamente su un cluster HPC (es. Slurm) per indicare quanti "job" inviare simultaneamente. In locale si comportano in modo quasi identico.

Eseguiamo il nostro semplice workflow:

```shell
snakemake --cores 1
```

Controllate la cartella: ora avete i file creati automaticamente.
Provate a rilanciare lo stesso comando: Snakemake vi dirà `Nothing to be done` perché i file sono già aggiornati.

## Concetti Avanzati di Snakemake e Python

### Checkpoint (Snakemake)

Esistono anche cose più avanzate.

Normalmente, Snakemake deve calcolare l'intero grafo delle dipendenze prima di eseguire anche un solo comando. Deve sapere esattamente quali file verranno prodotti e di quanti passaggi ha bisogno.
Tuttavia, in bioinformatica, alcuni tool generano un numero imprevisto di output (ad esempio, suddividere un genoma in un numero variabile di frammenti).
La direttiva `checkpoint` risolve questo problema. Quando Snakemake incontra un checkpoint:

1. Mette in pausa la costruzione del grafo
2. Esegue il comando
3. Ispeziona la cartella di output per vedere quanti e quali file sono stati effettivamente creati
4. Riprende la costruzione del grafo usando queste nuove informazioni.

### Wildcard Constraints (Snakemake)

Le wildcard (come `{sample}`) sono variabili che Snakemake cerca di riempire facendo combaciare i nomi dei file. A volte, però, rischiano di catturare i file sbagliati.
Usando `wildcard_constraints`, applichiamo delle Espressioni Regolari (Regex) per limitare i valori accettabili.

- L'espressione `\d+` significa "deve contenere solo numeri".
- L'espressione `[a-zA-Z]+` significa "deve contenere solo lettere".
  Questo meccanismo agisce da filtro, permettendo di smistare i file in percorsi di analisi differenti in modo automatico.

### Funzioni Standard di Python

Poiché Snakemake si basa su Python, possiamo usare librerie native per gestire la logica dei percorsi e dei file direttamente all'interno delle nostre regole.

- **`glob.glob(pattern)`**: una funzione che cerca nel disco tutti i file che corrispondono a un certo criterio (simile al comando `ls *.txt` nel terminale)
- **`os.path.join(percorso1, percorso2)`**: il modo corretto in Python per unire i percorsi dei file. Questa funzione costruisce il percorso in modo sicuro e indipendente dal sistema operativo in uso
- **`os.path.basename(percorso)`**: estrae solo il nome finale del file da un percorso completo
- **`.isdigit()` e `.isalpha()`**: sono metodi base delle stringhe in Python. Restituiscono `True` o `False`

### Expand (Funzione Nativa Snakemake)

La funzione `expand()` prende un modello di testo (contenente una o più variabili tra parentesi graffe) e una lista di valori, e genera automaticamente tutte le combinazioni possibili, tipo `expand("processed/text_{txt_id}.done", txt_id=['a', 'b'])`

### Esempio

Esempio complesso in cui:

- ha una regola checkpoint per creare un tot di file non conosciuti a priori
- se vede una variabile chiamata `{num_id}`, **devi** accettarla solo se contiene numeri (da 0 a 9)
- se vede una variabile chiamata `{txt_id}`, devi accettarla solo se contiene lettere (maiuscole o minuscole)
- funzioni python che aggreganpo i risultati

```python
import os
import glob

rule all:
    input:
        "numeric_report.txt",
        "text_report.txt"

rule generate_initial_data:
    output:
        "data.txt"
    shell:
        "echo 'Dati originali' > {output}"

checkpoint split_data:
    input:
        "data.txt"
    output:
        directory("chunks/")
    shell:
        """
        mkdir -p chunks/
        echo "Data 1" > chunks/chunk_1.txt
        echo "Data 2" > chunks/chunk_2.txt
        echo "Data 3" > chunks/chunk_3.txt
        echo "Data Alpha" > chunks/chunk_alpha.txt
        echo "Data Beta" > chunks/chunk_beta.txt
        """

rule process_numeric:
    input:
        "chunks/chunk_{num_id}.txt"
    output:
        "processed/numeric_{num_id}.done"
    wildcard_constraints:
        num_id="\d+"
    shell:
        "echo 'Processed numeric chunk: {wildcards.num_id}' > {output}"

rule process_text:
    input:
        "chunks/chunk_{txt_id}.txt"
    output:
        "processed/text_{txt_id}.done"
    wildcard_constraints:
        txt_id="[a-zA-Z]+"
    shell:
        "echo 'Processed text chunk: {wildcards.txt_id}' > {output}"

def aggregate_numeric(wildcards):
    checkpoint_dir = checkpoints.split_data.get(**wildcards).output[0]
    chunk_files = glob.glob(os.path.join(checkpoint_dir, "chunk_*.txt"))
    valid_ids = []

    for f in chunk_files:
        file_id = os.path.basename(f).split('_')[1].split('.')[0]
        if file_id.isdigit():
            valid_ids.append(file_id)

    return expand("processed/numeric_{num_id}.done", num_id=valid_ids)

def aggregate_text(wildcards):
    checkpoint_dir = checkpoints.split_data.get(**wildcards).output[0]
    chunk_files = glob.glob(os.path.join(checkpoint_dir, "chunk_*.txt"))
    valid_ids = []

    for f in chunk_files:
        file_id = os.path.basename(f).split('_')[1].split('.')[0]
        if file_id.isalpha():
            valid_ids.append(file_id)

    return expand("processed/text_{txt_id}.done", txt_id=valid_ids)

rule gather_numeric:
    input:
        aggregate_numeric
    output:
        "numeric_report.txt"
    shell:
        "cat {input} > {output}"

rule gather_text:
    input:
        aggregate_text
    output:
        "text_report.txt"
    shell:
        "cat {input} > {output}"
```

## Pipeline per variant calling

In questa lezione dovrete costruire una pipeline per fare quanto visto nelle lezioni precedenti.
Dovrete fare una pipeline per variant calling che utlizzi input lineare, come visto a lezione,
e read simulate. Fate un veloce confronto in tempo, spazio e qualità dei risultati del variant calling e dei grafi costruiti da esso.
Potenzialmente non serve nessuna delle cose avanzate viste ora
