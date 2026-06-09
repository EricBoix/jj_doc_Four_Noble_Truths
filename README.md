# Extracting some semantic structure out of the Four Noble Truths<!-- omit from toc -->

A tiny text to be used as sample for testing/debugging knowledge graph treatment pipelines.

## Table of contents<!-- omit from toc -->

- [Introduction](#introduction)
- [Running the full data workflow](#running-the-full-data-workflow)

## Introduction

This repository holds a very brief snippet of text (less than ten sentences), known as the ["Four Noble Truths"](https://en.wikipedia.org/wiki/Four_Noble_Truths), to be used as input to test various knowledge graph extraction data workflows.

The [chosen english version of the Four Noble Truths](250_BCE_-_Dhammacakkappavattana_Sutta_Four_Noble_Truths_Wikipedia_translation.md) was extracted from [Wikipedia's "Four Noble Truths" article](https://en.wikipedia.org/wiki/Four_Noble_Truths).

## Running the full data workflow

Prerequisite Knowledge Graph (KG) extraction: launch a neo4j database

```bash
cd `git rev-parse --show-toplevel`
docker build -t jejuness:jj_neo4j_docker https://github.com/EricBoix/jj_neo4j_docker.git
# Note: are we missing -e NEO4J_dbms_security_procedures_unrestricted: "apoc.*" \
docker run --rm --detach --name jj_neo4j_db \
    --publish=7474:7474 --publish=7687:7687 \
    --env NEO4J_AUTH=neo4j/your_password \
    -e NEO4J_apoc_export_file_enabled=true \
    -e NEO4J_apoc_import_file_enabled=true \
    -e NEO4J_apoc_import_file_use__neo4j__config=true \
    -v `pwd`/result_data/database:/data \
    jejuness:jj_neo4j_docker
```

Transmitting required info to following DB process users

```bash
echo "# Neo4j server designation and associated credentials" > .env
echo "NEO4J_URI=bolt://localhost:7687"                       >> .env
echo "NEO4J_USERNAME=neo4j"                                  >> .env
echo "NEO4J_PASSWORD=your_password"                          >> .env
```

Configure the KG (Knowledge Graph) extraction

```bash
if [ ! -f .env ]; then
  echo ".env file with DB info does not exist. Exiting."
  return
fi
# Configuring the llm model to be used
echo "### LLM server designation and associate credential"      >> .env
echo "MODEL_URL=https://ollama-ui.pagoda.liris.cnrs.fr/ollama/" >> .env
echo "API_KEY=sk-dde3f395ce1b4bfcb5c0d7f4c46b9c0c"              >> .env
echo "MODEL=llama3:70b"                                         >> .env
```

Run the (Knowledge Graph) extraction

```bash
docker build -t jejuness:jj_build_knowledge_graph https://github.com/EricBoix/jj_build_knowledge_graph.git#:DockerContext
docker run --rm --tty --name jj_build_knowledge_graph \
  --network host \
  -v `pwd`/original_data:/data \
  --env-file .env \
  jejuness:jj_build_knowledge_graph extracting_graph_semantic_chuncker.py --input_directory /data \
  --load_markdown_document 250_BCE_-_Dhammacakkappavattana_Sutta_Four_Noble_Truths_Wikipedia_translation.md 
```

Dump the database content for later usage

```bash
# Dumping requires the DB to be halted properly
docker stop $(docker ps -q --filter ancestor=jejuness:jj_neo4j_docker )
docker run --interactive --tty --rm  \
    --volume=`pwd`/result_data/database:/data \
    --volume=`pwd`/result_data/backups:/backups \
    neo4j/neo4j-admin neo4j-admin database dump neo4j --to-path=/backups
# Alas, when restoring the dump, the provided database name must have a 
# length between 1 and 63 characters...
mv result_data/backups/neo4j.dump result_data/backups/neo4j.Four-Noble-Truths-Wikipedia-translation.Markdown.dump
```

Restart the database (out of previous dump)...

```bash
rm -fr result_data/database     # WARNING: this deletes all your databases !
# The name of the dump file DOES matter: we have to restore it properly
mv result_data/backups/neo4j.Four-Noble-Truths-Wikipedia-translation.Markdown.dump result_data/backups/neo4j.dump
docker run --interactive --tty --rm \
    --volume=`pwd`/result_data/database:/data \
    --volume=`pwd`/result_data/backups:/backups \
    neo4j/neo4j-admin neo4j-admin database load neo4j.Four-Noble-Truths-Wikipedia-translation.Markdown --from-path=/backups

# Neo4j launching command is identical as the one done above
docker run --rm --detach --name jj_neo4j_db \
    --publish=7474:7474 --publish=7687:7687 \
    --env NEO4J_AUTH=neo4j/your_password \
    -e NEO4J_apoc_export_file_enabled=true \
    -e NEO4J_apoc_import_file_enabled=true \
    -e NEO4J_apoc_import_file_use__neo4j__config=true \
    -v `pwd`/result_data/database:/data \
    jejuness:jj_neo4j_docker
```

Eventually, extract knowledge graph [Turtle](https://en.wikipedia.org/wiki/Turtle_(syntax)) FILE

```bash
docker build -t jejuness:jj_neo4j_to_rdf_ttl https://github.com/EricBoix/jj_neo4j_to_rdf_ttl.git#:DockerContext
docker run --rm \
  --network host \
  -v `pwd`/result_data:/output \
  --env-file .env \
  jejuness:jj_neo4j_to_rdf_ttl \
  neo4j_to_rdf.py /output/graph.ttl
```

and turn off the neo4j db

```bash
docker stop $(docker ps -q --filter ancestor=jejuness:jj_neo4j_docker )
```
