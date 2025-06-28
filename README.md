# 🧬 OrthoMCL Pipeline for Two Genomes

This guide documents the full process to run **OrthoMCL v2.0.9** for orthologous gene detection between two genomes using local BLAST and MySQL.

---

## 📁 Project Structure

```
OrthoMCL/
├── orthomclSoftware-v2.0.9/         # OrthoMCL installation directory
├── BD21/                            # Working directory
│   ├── formatted_files/             # Reformatted FASTA inputs
│   │   ├── BGR1.fasta
│   │   └── BD21.fasta
```

---

## 📦 Prerequisites

- `conda` or system installation of:
  - `blast+` (makeblastdb, blastp)
  - `mysql-server` (version 5.7+)
  - `perl` with required CPAN modules
  - `OrthoMCL v2.0.9`
  - `mcl` clustering tool

---

## ⚙️ Step 1: Format FASTA Headers

Format each protein FASTA header like:

```
>SpeciesCode|GeneID
```

**Example:**

```fasta
>BGR1|WP_004186391.1
MADEUPSEQUENCE...
```

This format is mandatory for OrthoMCL to identify species origin in multi-genome input.

---

## 🧹 Step 2: Filter Input FASTA Files

Use `orthomclFilterFasta` to remove short or poor-quality sequences:

```bash
/home/ismam/NCBI_LOCAL_BLAST/OrthoMCL/orthomclSoftware-v2.0.9/bin/orthomclFilterFasta /home/ismam/NCBI_LOCAL_BLAST/OrthoMCL/BD21/formatted_files/ 10 20
```

- `10`: minimum protein length
- `20`: maximum % stop codons allowed

**Output:**

- `goodProteins.fasta`
- `poorProteins.fasta` (rejected)

---

## 🔍 Step 3: All-vs-All BLASTP

Prepare a BLAST database and run self-BLAST:

```bash
makeblastdb -in goodProteins.fasta -dbtype prot

blastp -query goodProteins.fasta \
       -db goodProteins.fasta \
       -out all_vs_all.blast \
       -outfmt 6 \
       -evalue 1e-5 \
       -num_threads 8
```

---

## 🧾 Step 4: Parse BLAST Output

```bash
orthomclBlastParser all_vs_all.blast formatted_files/ > similarSequences.txt
```

This prepares the BLAST results for OrthoMCL database loading.

---

## 🗄️ Step 5: Setup MySQL Database

### a. Launch MySQL

```bash
mysql -u root -p
```

### b. Run the following SQL commands

```sql
CREATE DATABASE orthomcl;

CREATE USER 'USERNAME'@'localhost' IDENTIFIED BY 'YOURPASSWORD';

GRANT ALL PRIVILEGES ON orthomcl.* TO 'orthomcluser'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

---

## 📄 Step 6: Configure OrthoMCL

Copy and edit the configuration template:

```bash
cp orthomcl.config.template orthomcl.config
```

Edit `orthomcl.config`:

```ini
dbVendor=mysql
dbConnectString=dbi:mysql:orthomcl
dbLogin=USERNAME
dbPassword=YOURPASSWORD
similarSequences=similarSequences.txt
orthologTable=ortholog
```

---

## 🛠️ Step 7: Install Schema and Load Data

```bash
orthomclInstallSchema orthomcl.config

orthomclLoadBlast orthomcl.config similarSequences.txt

orthomclPairs orthomcl.config pairs.log cleanup=yes
```

---

## 📤 Step 8: Dump Pairs for MCL

```bash
orthomclDumpPairsFiles /home/ismam/NCBI_LOCAL_BLAST/OrthoMCL/orthomclSoftware-v2.0.9/bin/orthomcl.config
```

This creates `mclInput`, the input file for clustering.

---

## 🔄 Step 9: Run MCL Clustering

```bash
mcl mclInput --abc -I 1.5 -o mclOutput
```

- `-I 1.5`: inflation parameter (adjustable)

---

## 📦 Step 10: Convert to Orthologous Groups

```bash
orthomclMclToGroups groups 1 < mclOutput > groups.txt
```

---

## ✅ Output

`groups.txt` will contain lines like:

```
ORTHOMCL1(BGR1|WP_004186391.1 BD21|WP_004100000.1)
ORTHOMCL2(BGR1|WP_004186392.1 BD21|WP_004100001.1)
...
```

Each line represents one orthologous group.

---

## 🧠 Notes

- All FASTA headers must use the `SpeciesID|GeneID` format.
- Ensure your MySQL server is running and accessible.
- Adjust the MCL inflation parameter depending on how tight you want the clusters.
- This pipeline scales to any number of genomes with similar steps.

---

## 🔗 References

- [OrthoMCL Documentation](https://orthomcl.org/)
- [BLAST+ Guide](https://www.ncbi.nlm.nih.gov/books/NBK279690/)
- [MCL Algorithm](https://micans.org/mcl/)

---

## 👨‍🔬 Author

Ismam Ahmed Protic  
University of Nevada, Reno
