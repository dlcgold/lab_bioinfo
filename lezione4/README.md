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

Esecuzione parallela (Core vs Jobs)\*\*
Lanciamo la pipeline. Esistono due flag principali per le risorse:

- `--cores N` (o `-c N`): Quanti core locali del vostro PC Snakemake può usare al massimo per parallelizzare.
- `--jobs N` (o `-j N`): Simile a `cores`, ma usato tipicamente su un cluster HPC (es. Slurm) per indicare quanti "job" inviare simultaneamente. In locale si comportano in modo quasi identico.

Eseguiamo il nostro semplice workflow:

```shell
snakemake --cores 1
```

Controllate la cartella: ora avete i file creati automaticamente.
Provate a rilanciare lo stesso comando: Snakemake vi dirà `Nothing to be done` perché i file sono già aggiornati.

## Pipeline per variant calling

In questa lezione dovrete costruire una pipeline per fare quanto visto nelle lezioni precedenti.
Dovrete fare una pipeline per variant calling che utlizzi input lineare, come visto a lezione,
e read simulate. Fate un veloce confronto in tempo, spazio e qualità dei risultati del variant calling e dei grafi costruiti da esso.
