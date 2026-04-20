# Lezione 3

# Lezione 3: Dai Genomi Lineari ai Genomi a Grafo (Pangenomica)

Nella lezione precedente abbiamo usato un genoma di riferimento lineare. Tuttavia, questo approccio ha un limite: ignora completamente la diversità genetica della popolazione (reference-bias).
La soluzione moderna è la pangenomica: rappresentando multipli genomi, ad esempio, con un grafo.

Per prima cosa, installiamo i tool necessari nel nostro ambiente:

```shell
mamba install -c bioconda vg tabix -y
```

Per la costruzione di grafi abbiamo vari tool, che costruiscono grafi diversi partendo da input diversi:

- **[minigraph](https://github.com/lh3/minigraph)**

  - **Input:** MSA di vari genomi.
  - **Tipo di grafo:** Aggiunge al grafo solo le grandi Varianti Strutturali (> 50bp). Ignora completamente gli SNP e perde i _path_ completi (sequence graph).

- **[minigraph-cactus](https://github.com/ComparativeGenomicsToolkit/cactus)**

  - **Input:** MSA di vari genomi..
  - **Tipo di grafo:** Usa `minigraph` per creare rapidamente lo scheletro e poi lo arricchisce usando un consenso come reference. Conserva tutte le varianti (SNP/varianti strutturali) e tutti i _path_ (variation graph).

- **[pggb](https://github.com/pangenome/pggb)** _(PanGenome Graph Builder)_

  - **Input:** MSA di vari genomi.
  - **Tipo di grafo:** Produce un pangenoma senza l'uso di un consenso, conservando tutte le varianti e i _path_. Computazionalmente molto più esigente.

- **[VG](https://github.com/vgteam/vg)** _(Variation Graph Toolkit)_
  - **Input:** Un genoma di riferimento lineare (FASTA) e un catalogo di varianti note (VCF).
  - **Tipo di grafo:** **Grafo VCF-based (VG/GFA).** Costruisce uno scheletro lineare e crea delle "bolle" per ogni mutazione annotata nel file VCF. Non scopre _nuove_ varianti strutturali complesse da genomi interi, ma rappresenta quelle del VCF.

Oggi vedremo solamente VG per mancanza di tempo

Per prima cosa indicizziamo le varianti della lezione precedente:

```
bcftools view variants.bcf | bgzip -o variants.vcf.gz
tabix -p vcf variants.vcf.gz
```

Usiamo il comando `construct` per costruire il grafo:

```shell
vg construct -r lambda.fa -v variants.vcf.gz > graph.vg
```

VG usa un formato proprio (encoda il grafo con protobuf) ma possiamo convertire il risultato in formato GFA:

```
vg view graph.vg | most
vg view graph.vg > graph.gfa
```

Come l'altra volta possiamo vedere qualche statistica:

```shell
vg stats -zlLs graph.vg
```

Possiamo anche visualizzare il grafo con tool esterni, tipo [Bandage](https://github.com/rrwick/Bandage.git).

Come per il caso lineare possiamo mappare le nostre read contro il grafo.

- **[vg map](https://github.com/vgteam/vg/wiki/Working-with-a-whole-genome-variation-graph)**

  - **Input:** Short/long read e un grafo indicizzato.
  - **Tipologia di Allineamento:** Allineatore general-purpose di `vg`. Cerca di allineare le read esplorando tutti percorsi all'interno del grafo.

- **[vg mpmap](https://github.com/vgteam/vg/wiki/Multipath-alignments-and-vg-mpmap)**

  - **Input:** Short/long read e un grafo indicizzato.
  - **Tipologia di Allineamento:** Allineamenti "multi-percorso" (multipath) che catturano l'incertezza locale. È fondamentale in ambiti come la pantrascriptomica.

- **[vg giraffe](https://github.com/vgteam/vg/wiki)**

  - **Input:** Short reads e grafo indicizzato.
  - **Tipologia di Allineamento:** Miglioramento di vg map.

- **[GraphAligner](https://github.com/maickrau/GraphAligner)**

  - **Input:** Short/long read e un grafo (senza path) non indicizzato.
  - **Tipologia di Allineamento:** Sviluppato specificamente per long read e sequence graph.

Indicizziamo il grafo:

```shell
vg index -x graph.xg -g graph.gcsa -k 16 graph.vg
```

E allineiamo le short reads simulate la scorsa lezione:

```shell
vg map -d graph -f short_read1.fq > aln.gam
```

Anche i questo caso possiamo analizzare il risultato degli allineamenti:

```shell
vg convert --gam-to-gaf aln.gam graph.vg | most
vg stats -a aln.gam
```

## Extra

Provate a testare altri metodi di costruzione di grafi del pangenoma e di allineamento.
