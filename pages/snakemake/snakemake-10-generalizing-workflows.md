It's generally a good idea to separate project-specific parameters from the
actual implementation of the workflow. If we want to move all project-specific
information to `config.yml`, and let the Snakefile be a more general RNA-seq
analysis workflow, we need the config file to:

* Specify which samples to run.
* Specify which genome to align to and where to download its sequence and
  annotation files.
* (Any other parameters we might need to make it into a general workflow,
  *e.g.* to support both paired-end and single-read sequencing)

> **Note** <br>
> Putting all configuration in `config.yml` will break the
> `generate_rulegraph` rule. You can fix it either by replacing
> `--config max_reads=0` with `--configfile=config.yml` in the shell
> command of that rule in the Snakefile, or by adding
> `configfile: "config.yml"` to the top of the Snakefile (as mentioned
> in a tip above).

The first point is straightforward; rather than using `SAMPLES = ["..."]` in
the Snakefile we define it as a parameter in `config.yml`. Do a dry-run
afterwards to make sure that everything works as expected. You can either add
it as a list similar to the way it was expressed before by adding
 `SAMPLES: ["..."]` to `config.yml`, or you can use this yaml notation:

```yaml
sample_ids:
- SRR935090
- SRR935091
- SRR935092
```

The second point is trickier. Writing workflows in Snakemake is quite
straightforward when the logic of the workflow is reflected in the file names,
*i.e.* `my_sample.trimmed.deduplicated.sorted.fastq`, but that isn't always the
case. In our case we have the FTP paths to the genome sequence and annotation
where the naming doesn't quite fit with the rest of the workflow. The easiest
solution is probably to make three parameters to hold these values, say
`genome_id`, `genome_fasta_path` and `genome_gff_path`, but we will go for
a somewhat more complex but very useful alternative. We want to construct
a dictionary where something that will be a wildcard in the workflow is the key
and the troublesome name is the value. An example might make this clearer (this
is also in `config.yml`). This is a nested dictionary where "genomes" is a key
with another dictionary as value, which in turn has genome ids as keys and so
on. The idea is that we have a wildcard in the workflow that takes the id of
a genome as value (either "NCTC8325" or "ST398" in this case). The fasta and
gff3 paths can then be retrieved based on the value of the wildcard.

```yaml
genomes:
  NCTC8325:
    fasta: ftp://ftp.ensemblgenomes.org/pub/bacteria/release-37/fasta/bacteria_18_collection/staphylococcus_aureus_subsp_aureus_nctc_8325/dna//Staphylococcus_aureus_subsp_aureus_nctc_8325.ASM1342v1.dna_rm.toplevel.fa.gz
    gff3: ftp://ftp.ensemblgenomes.org/pub/bacteria/release-37/gff3/bacteria_18_collection/staphylococcus_aureus_subsp_aureus_nctc_8325//Staphylococcus_aureus_subsp_aureus_nctc_8325.ASM1342v1.37.gff3.gz
  ST398:
    fasta: ftp://ftp.ensemblgenomes.org/pub/bacteria/release-37/fasta/bacteria_18_collection//staphylococcus_aureus_subsp_aureus_st398/dna/Staphylococcus_aureus_subsp_aureus_st398.ASM958v1.dna.toplevel.fa.gz
    gff3: ftp://ftp.ensemblgenomes.org/pub/bacteria/release-37/gff3/bacteria_18_collection/staphylococcus_aureus_subsp_aureus_st398//Staphylococcus_aureus_subsp_aureus_st398.ASM958v1.37.gff3.gz
```

Let's now look at how to do the mapping from genome id to fasta path in the
rule `get_genome_fasta`. This is how the rule currently looks (if you have
added the log section as previously described).

```python
rule get_genome_fasta:
    """
    Retrieve the sequence in fasta format for a genome.
    """
    output:
        "data/raw_external/NCTC8325.fa.gz"
    log:
        "results/logs/get_genome_fasta/NCTC8325.log"
    shell:
        """
        wget ftp://ftp.ensemblgenomes.org/pub/bacteria/release-37/fasta/bacteria_18_collection/staphylococcus_aureus_subsp_aureus_nctc_8325/dna//Staphylococcus_aureus_subsp_aureus_nctc_8325.ASM1342v1.dna_rm.toplevel.fa.gz -O {output} -o {log}
        """
```

We don't want the hardcoded genome id, so replace that with a wildcard, say
`{genome_id}`. In general, we want the rules as far downstream as possible in
the workflow to be the ones that determine what the wildcards should resolve
to. In our case this is `align_to_genome`. You can think of it like the rule
that really "needs" the file asks for it, and then it's up to Snakemake to
determine how it can use all the available rules to generate it. Here we say "I
need this genome index to align my sample to" and it's up to Snakemake to
determine how to download and build the index.

We're almost done, but there is one more tricky thing left. Take a look below
at the `params` section. Here we have defined a function to generate the
parameter `fasta_path`. This is what's called an anonymous function, or lambda
expression, but any normal function would work as well. What happens is that
the function takes the `wildcards` object as its only argument. This object
allows access to the wildcards values via attributes (here
`wildcards.genome_id`). The function will then look in the nested `config`
dictionary and return the value of the fasta path for the key
`wildcards.genome_id`. Neat!

```python
rule get_genome_fasta:
    """
    Retrieve the sequence in fasta format for a genome.
    """
    output:
        "data/raw_external/{genome_id}.fa.gz"
    params:
        fasta_path = lambda wildcards: config["genomes"][wildcards.genome_id]["fasta"]
    log:
        "results/logs/get_genome_fasta/{genome_id}.log"
    shell:
        """
        wget {params.fasta_path} -O {output} -o {log}
        """
```

Now change in `get_genome_gff3` in the same way.

Also change in `index_genome` to use a wildcard rather than a hardcoded genome
id. Here you will run into a complication if you have followed the previous
instructions and use the `expand()` expression. We want the list to expand to
`["intermediate/{genome_id}.1.bt2", "intermediate/{genome_id}.2.bt2", ...]`,
*i.e.* only expanding the wildcard referring to the bowtie2 index. To keep the
`genome_id` wildcard from being expanded we have to "mask" it with double curly
brackets: `{{genome_id}}`.

Lastly, we need to define somewhere which genome id we actually want to use.
This needs to be done both in `align_to_genome` and `generate_count_table`.
Do this by introducing a parameter in `config.yml` called `"genome_id"`. See
below for an example for `align_to_genome`. Here the `substr` wildcard gets
expanded from a list while `genome_id` gets expanded from the config file.

```python
output:
    expand("intermediate/{genome_id}.{substr}.bt2",
           genome_id = config["genome_id"],
           substr = ["1", "2", "3", "4", "rev.1", "rev.2"])
```

> **Summary** <br>
> Well done! You now have a complete Snakemake workflow with a number of
> excellent features:
>
> - A general RNA-seq pipeline which can easily be reused between projects,
>   thanks to clear separation between code and settings.
> - Great traceability due to logs and summary tables.
> - Clearly defined the environment for the workflow using Conda.
> - The workflow is neat and free from temporary files due to using `temp()` and
>   `shadow`.
> - A logical directory structure which makes it easy to separate raw data,
>   intermediate files, and results.
> - A project set up in a way that makes it very easy to distribute and
>   reproduce either via Git, Snakemake's `--archive` option or a Docker image.

> **Quick recap** <br>
> In this section we've learned:
>
> - How to generalize a Snakemake workflow.
