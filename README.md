# 📖 Guidelines for SPARQL endpoints metadata

Add precomputed metadata to your SPARQL endpoint to make it easier to query it by humans and machines:

- **SPARQL examples** give a good idea of the capabilities and known use-cases of your endpoint, as well as recommended query patterns. Most public endpoints already provide example queries, we propose to expose them directly in the SPARQL endpoint in a standard format.
- **A lightweigth classes schema** provide an overview of all classes actually present in the endpoint and the predicates they use.

<details><summary>About the choice of standard used to define the classes schema</summary>

- [ ] Ontologies are not schema. Ontologies describe *possible* concepts. A schema here describes the *actual content* of the endpoint.
- [ ] ShEx/SHACL shapes are too detailed, constraints cannot automatically be inferred by querying the endpoint, and, more important, they do not enable to communicate counts of classes and predicates (e.g. there are 200 unique countries in this endpoint).
- [x] We chose to use the [Vocabulary of Interlinked Datasets (VoID) description](https://www.w3.org/TR/void/), which was also recommended by the [Healthcare and Life Science (HCLS) community profile draft](https://www.w3.org/TR/hcls-dataset/).

</details>

> [!CAUTION]
>
> Most endpoints are too large to compute the classes schema on-the-fly, and doing so repeatedly would be computationally, and ecologically, very expensive, so it needs to be precomputed.

## 🧑‍🍳 How?

Precompute the metadata as RDF, and upload it to the endpoint, either:

- Directly in the endpoint, usually in a named graph ending with `/.well-known/sparql-examples`
  - Or in the endpoint’s service description	


> [!IMPORTANT]
>
> Putting the metadata in the endpoint has many advantages:
>
> - The endpoint URL is all a user needs: any user or systems can directly retrieve the metadata from the endpoint, no need to query an external service to get the metadata
> - Cheap and efficient stack, no need to deploy and maintain an extra service and/or database
> - Can be queried from any client using a standard HTTP request, no need for adhoc niche packages

The following ontologies are used to define the metadata:

- [SHACL ontology](https://www.w3.org/TR/shacl/) to represent SPARQL query examples
- [VoID ontology](https://www.w3.org/TR/void/) to represent the classes schema
- [VoID-ext](http://ldf.fi/void-ext#), an extension to enable describing data properties in VoID (predicates that point to a value, like a string or int, instead of another node IRI)

## 💻 Apps using this framework

Example of systems relying on this metadata framework:

- A chat app to help users write SPARQL queries for all SIB endpoints: [expasy.org/chat](https://expasy.org/chat)
- A query editor with context-aware autocomplete: [sib-swiss.github.io/sparql-editor](https://sib-swiss.github.io/sparql-editor/)

## 🚀 Setup

Download the latest release of the 2 jar files used to compile metadata:

```sh
curl -s https://api.github.com/repos/sib-swiss/sparql-examples-utils/releases/latest \
  | grep "browser_download_url" \
  | grep "uber.jar" \
  | cut -d '"' -f 4 \
  | xargs curl -LO
curl -s https://api.github.com/repos/sib-swiss/void-generator/releases/latest \
  | grep "browser_download_url" \
  | grep "uber.jar" \
  | cut -d '"' -f 4 \
  | xargs curl -LO
mkdir -p data
```

### 📑 Document examples

Document query examples for your SPARQL endpoint in RDF turtle files, and use the [`sparql-examples-utils.jar`](https://github.com/sib-swiss/sparql-examples-utils) CLI to validate and compile all files.

1. Fork this repository, or copy its `examples/` folder in your project 
2. Edit the content of the `examples/` folder:
   1. Each subfolder corresponds to a different endpoint
   2. Record prefixes in the `prefixes.ttl` files
   3. Prefixes in `examples/prefixes.ttl` are common to all endpoints, while the `prefixes.ttl` file in each subfolder is specific to the corresponding endpoint
   4. Copy an existing example and adapt it to get started

Compile all query files for one endpoint into a RDF turtle file including prefix declarations (stored in the `data/` folder):

```sh
java -jar sparql-examples-utils-*.jar convert -i examples/ -p UniProt -f ttl > data/examples.ttl
```

Or compile all endpoints as JSON-LD to the standard output:

```sh
java -jar sparql-examples-utils-*.jar convert -i examples/ -p all -f jsonld
```

> [!TIP]
>
> See the [sib-swiss/sparql-examples](https://github.com/sib-swiss/sparql-examples) repository for more details.

### 🧮 Compute classes schema

Run the [`void-generator`](https://github.com/sib-swiss/void-generator) for your endpoint:

```sh
java -jar void-generator-*.jar -r https://sparql.wikipathways.org/sparql \
   -p https://sparql.wikipathways.org/sparql \
   --void-file data/void-wikipathway.ttl \
   --iri-of-void 'https://rdf.wikipathway.org/.well-known/void#' \
   -g http://rdf.wikipathways.org/
```

Depending on the size and structure of your endpoint, generating the classes schema may take a long time (up to hours). You can optimize the process using options such as:

- `--filter-expression-to-exclude-classes-from-void`: exclude non-meaningful classes (e.g. very sparse ontology classes). Variable should be `?clazz`

  - Example excluding CHEBI classes:

    ```sh
    --filter-expression-to-exclude-classes-from-void "!STRSTARTS(STR(?clazz), 'http://purl.obolibrary.org/obo/CHEBI_')"
    ```

- `--optimize-for`: select triplestore query optimizations (Virtuoso, QLever, or default SPARQL)

- `--count-distinct-subjects false`

- `--count-distinct-objects false`

Example command optimized for a local Virtuoso endpoint using its JDBC connector:

```sh
java -jar void-generator-*.jar \
    --user dba \
    --password dba \
    --virtuoso-jdbc=jdbc:virtuoso://localhost:1111/charset=UTF-8 \ # note the localhost and "isql-t" port
    -r "https://YOUR_SPARQL_ENDPOINT/sparql" \
    -s data/void-file-locally-stored.ttl \
    -i "https://YOUR_SPARQL_ENDPOINT/.well-known/void"
```

> [!TIP]
>
> See the [sib-swiss/void-generator](https://github.com/sib-swiss/void-generator) repository for more details.

### 📤 Upload the metadata to your endpoint

Once the RDF files are generated, upload them either to your endpoint or its service description.

Uploading to the endpoint is simpler, just load them into a named graph, but mixes metadata and data. Some administrators prefer a clean separation.

Exposing metadata through the service description is ideal (and aligns with its intended purpose), but most triplestores do not allow editing it directly. In that case, precompute the service description RDF and serve it via a proxy rule when the endpoint is queried without a `query` parameter.

> [!TIP]
>
> If you are looking for a system to automatize uploading to SPARQL endpoints, we recommend to look into [`kgsteward`](https://github.com/sib-swiss/kgsteward).

### 🪏 Retrieve the metadata

Example SPARQL queries to retrieve these metadata

Retrieve query examples:

```SPARQL
PREFIX sh: <http://www.w3.org/ns/shacl#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX spex: <https://purl.expasy.org/sparql-examples/ontology#>

SELECT DISTINCT ?sq ?comment ?query
WHERE {
    ?sq a sh:SPARQLExecutable ;
        rdfs:comment ?comment ;
        sh:select|sh:ask|sh:construct|spex:describe ?query .
} ORDER BY ?sq
```

Classes schema  without subject/objects count:

```SPARQL
PREFIX up: <http://purl.uniprot.org/core/>
PREFIX void: <http://rdfs.org/ns/void#>
PREFIX void-ext: <http://ldf.fi/void-ext#>

SELECT DISTINCT ?subjectClass ?prop ?objectClass ?objectDatatype
WHERE {
  {
    ?cp void:class ?subjectClass ;
        void:propertyPartition ?pp .
    ?pp void:property ?prop .
    OPTIONAL {
        {
            ?pp  void:classPartition [ void:class ?objectClass ] .
        	
        } UNION {
            ?pp void-ext:datatypePartition [ void-ext:datatype ?objectDatatype ] .
        }
    }
  } UNION {
    ?linkset void:subjectsTarget [ void:class ?subjectClass ] ;
      void:linkPredicate ?prop ;
      void:objectsTarget [ void:class ?objectClass ] .
  }
}
```

Classes schema with subject/objects count:

```SPARQL
PREFIX up: <http://purl.uniprot.org/core/>
PREFIX void: <http://rdfs.org/ns/void#>
PREFIX void-ext: <http://ldf.fi/void-ext#>

SELECT DISTINCT ?subjectsCount ?subjectClass ?prop ?objectClass ?objectsCount ?objectDatatype
WHERE {
  {
    ?cp void:class ?subjectClass ;
        void:entities ?subjectsCount ;
        void:propertyPartition ?pp .
    ?pp void:property ?prop .
    OPTIONAL {
        {
            ?pp  void:classPartition [ void:class ?objectClass ; void:triples ?objectsCount ] .
        } UNION {
            ?pp void-ext:datatypePartition [ void-ext:datatype ?objectDatatype ] .
        }
    }
  } UNION {
    ?linkset void:subjectsTarget [ void:class ?subjectClass ; void:entities ?subjectsCount ] ;
      void:linkPredicate ?prop ;
      void:objectsTarget [ void:class ?objectClass ; void:entities ?objectsCount ] .
  }
}
```