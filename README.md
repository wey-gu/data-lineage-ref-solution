# A reference datalab stack

This reference project will set up an end to end minimal Data Stack with playground environment coverring from Data Warehouse/Database, ETL Pipeline, Data BI, to Metadata Discovery/Lineage Management System.
I started this for learning purposes and I hope it helps to new data engineers understand how modern data infra with the open-source community's joint efforts could be used to build a better data science ecosystem.

There are already a bunch of projects out there that covers most of the stack already, my motivation is to **add a layer of Metadata Discovery/Lineage Management System** to the stack. And from this diagram we could see I put most of the components as Metadata Sources:

![](https://user-images.githubusercontent.com/1651790/168849779-4826f50e-ff87-4e78-b17f-076f91182c43.svg)

## The Data Stack

Let's start with the Data Stack.

### Database and Data Warehouse

For processing and consuming raw and intermediate data, one or more databases and/or warehouses should be used.

It could be any DB/DW like Hive, Apache Delta, TiDB, Cassandra, MySQL, or Postgres, in this reference project, we simply choose one of the most popular ones: Postgres. And our reference lab comes with the first service:

✅ - Data warehouse: Postgres

### DataOps

We should have some sort of DataOps setup to enable pipelines and environments to be repeatable, testable, and version-controlled.

Here, we used [Meltano](https://gitlab.com/meltano/meltano) created by GitLab.

Meltano is a just-work DataOps platform that connected [Singer](https://singer.io/) as the EL and [dbt](https://getdbt.com/) as the T in a magically elegant way, it is also connected to some other dataInfra utilities such as Apache Superset and Apache Airflow, etc.

Thus, we have one more thing to be included:

✅ - GitOps: Meltano

### ETL

And under the hood, we will E(extract) and L(load) data from many different data sources to data targets leveraging [Singer](https://singer.io/) together with Meltano, and do T(transformation) with [dbt](https://getdbt.com/).

✅ - EL: Singer

✅ - T: dbt

### Data Visualization

How about creating dashboards, charts, and tables for getting the insights into all the data?

![](https://superset.apache.org/img/explorer5.jpg)

[Apache Superset](https://superset.apache.org/) is one of the greatest visualization platforms we could choose from, and we just add it to our packet!

✅ - Dashboard: Apache Superset

### Job Orchestration

In most cases, our DataOps jobs grow to the scale to be executed in a long time that needs to be orchestrated, and here comes the [Apache Airflow](https://airflow.apache.org/).

✅ - DAG: Apache Airflow

### Metadata governance

With more components and data being introduced to a data infra, there will be massive metadata in all lifecycle of databases, tables, schemas, dashboards, DAGs, applications, and their administrators and teams could be collectively managed, connected, and discovered.

[Linux Foundation Amundsen](https://www.amundsen.io/amundsen/) is one of the best projects solving this problem.

![](https://www.amundsen.io/amundsen/frontend/docs/img/table_detail_page.png)



✅ - Data Discovery: Linux Foundation Amundsen

With a graph database as the source of truth to accelerate the multi-hop queries together with elasticsearch as the full-text search engine, Amundsen indexes all the metadata and their lineage smoothly, and beautifully in the next level.

By default, [neo4j](https://neo4j.org/) was used as the graph database, while I will be using [Nebula Graph](http://nebula-graph.io/) instead in this project due to I am more familiar with the latter.

✅ - Full-text Search: elasticsearch

✅ - Graph Database: Nebula Graph

Now, with the components in our stack being revealed, let's have them assembled.

## Environment Bootstrap, Component overview

We will try our best to make things clean and isolated. It's assumed you are running on a UNIX-like system with internet and Docker Compose being installed.

> Please refer [here](https://docs.docker.com/compose/install/) to install Docker and Docker Compose before moving forward.

I am running it on Ubuntu 20.04 LTS X86_64, but there shouldn't be issues on other distros or versions of Linux.

### Run a Data Warehouse/ Database

First, let's install Postgres as our data warehouse.

This oneliner will help create a Postgres running in the background with docker, and when being stopped it will be cleaned up(`--rm`).

```bash
docker run --rm --name postgres \
    -e POSTGRES_PASSWORD=lineage_ref \
    -e POSTGRES_USER=lineage_ref \
    -e POSTGRES_DB=warehouse -d \
    -p 5432:5432 postgres
```

Then we could verify it with Postgres CLI or GUI clients.

> Hint: You could use VS Code extension: [SQL tools](https://marketplace.visualstudio.com/items?itemName=mtxr.sqltools) to quickly connect to multiple RDBMS(MariaDB, Postgres, etc.) or even Non-SQL DBMS like Cassandra in a GUI fashion.

### Setup DataOps toolchain for ETL

Then, let's get Meltano with Singler and dbt installed.

Meltano helps us manage ETL utilities(as plugins) and all of their configurations(the pipelines). Those meta-information sits in meltano configurations and its [system database](https://docs.meltano.com/concepts/project#system-database), where the configurations are file-based(could be managed with git) and by default the system database is SQLite.

#### Installation of Meltano

The workflow using Meltano is to initiate a `meltano project` and start to add E, L, and T into the configuration files. The initiation of a project just requires a CLI command call: `meltano init yourprojectname` and to do that, we could install Meltano either with Python's package manager: pip or via a Docker image:

- Install Meltano with pip in a python virtual env:

```bash
mkdir .venv
# example in a debian flavor Linux distro
sudo apt-get install python3-dev python3-pip python3-venv python3-wheel -y
python3 -m venv .venv/meltano
source .venv/meltano/bin/activate
python3 -m pip install wheel
python3 -m pip install meltano

# init a project
mkdir meltano_projects && cd meltano_projects
# replace <yourprojectname> with your own one
touch .env
meltano init <yourprojectname>
```

- "Install" Meltano via Docker

```bash
docker pull meltano/meltano:latest
docker run --rm meltano/meltano --version

# init a project
mkdir meltano_projects && cd meltano_projects

# replace <yourprojectname> with your own one
touch .env
docker run --rm -v "$(pwd)":/projects \
             -w /projects --env-file .env \
             meltano/meltano init <yourprojectname>
```

Apart from `meltano init`, there are a couple of other commands like `meltano etl` to perform ETL executions, and `meltano invoke <plugin>` to call plugins' command, always check the [cheatsheet](https://docs.meltano.com/reference/command-line-interface) for quick referencing.

#### The Meltano UI

Meltano also comes with a web-based UI, to start it, just run:

```bash
meltano ui
```

Then it's listening to http://localhost:5000.

For Docker, just run the container with the 5000 port exposed, here we didn't provide `ui` in the end due to the container's default command being `meltano ui` already.

```bash
docker run -v "$(pwd)":/project \
             -w /project \
             -p 5000:5000 \
             meltano/meltano
```

#### Example Meltano projects

When writing this article, I noticed that [Pat Nadolny](https://github.com/pnadolny13) had created [great examples](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/singer_dbt_jaffle) on an example dataset for Meltano with dbt(And with [Airflow](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/dbt_orchestration) and [Superset](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/jaffle_superset), too!). We will not recreate the examples and use Pat's great ones.

> Note that Andrew Stewart had created another one with a slightly older version of configuration files.

You could follow [here](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/singer_dbt_jaffle) to run a pipeline of:

- [tap-CSV](https://hub.meltano.com/taps/csv)(Singer), extracting data from CSV files
- [target-postgres](https://hub.meltano.com/targets/postgres)(Singer), loading data to Postgres
- [dbt](https://hub.meltano.com/transformers/dbt), transform the data into aggregated tables or views

> You should omit the step of running the local Postgres with docker as we had already created one, be sure to change the Postgres user and password in `.env`.
>
> And it's basically as this(with meltano being installed as above):
>
> ```bash
> git clone https://github.com/pnadolny13/meltano_example_implementations.git
> cd meltano_example_implementations/meltano_projects/singer_dbt_jaffle/
> 
> meltano install
> touch .env
> echo PG_PASSWORD="lineage_ref" >> .env
> echo PG_USERNAME="lineage_ref" >> .env
> 
> # Extract and Load(with Singer)
> meltano run tap-csv target-postgres
> 
> # Trasnform(with dbt)
> meltano run dbt:run
> 
> # Generate dbt docs
> meltano invoke dbt docs generate
> 
> # Serve gnerated dbt docs
> meltano invoke dbt docs serve
> 
> # Then visit http://localhost:8080
> ```

Now, I assumed you had finished trying out `singer_dbt_jaffle` following its [README.md](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/singer_dbt_jaffle), and we could connect to the Postgres to see the loaded and transformed data being reflected as follow, the screenshot is from the SQLTool of VS Code:

![](https://user-images.githubusercontent.com/1651790/167540494-01e3dbd2-6ab1-41d2-998e-3b79f755bdc7.png)

### Setup a BI Platform for Dashboard

Now, we have the data in data warehouses, with ETL toolchains to pipe different data sources into it. How could those data be consumed?

BI tools like the dashboard could be one way to help us get insights from the data.

With Apache Superset, dashboards, and charts based on those data sources could be created and managed smoothly and beautifully.

The focus of this project was not on Apache Superset itself, thus, we simply reuse examples that [Pat Nadolny](https://github.com/pnadolny13) had created in [Superset as a utility if meltano Example](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/jaffle_superset).

#### Bootstrap Meltano and Superset

Create a python venv with Meltano installed:

```bash
mkdir .venv
python3 -m venv .venv/meltano
source .venv/meltano/bin/activate
python3 -m pip install wheel
python3 -m pip install meltano
```

Following [Pat's guide](https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/jaffle_superset), with tiny modifications:

- Clone the repo, enter the `jaffle_superset` project

```bash
git clone https://github.com/pnadolny13/meltano_example_implementations.git
cd meltano_example_implementations/meltano_projects/jaffle_superset/
```

- Modify the meltano configuration files to let Superset connect to the Postgres we created:

```bash
vim meltano_projects/jaffle_superset/meltano.yml
```

In my example, I changed the hostname to `10.1.1.111`, which is the IP of my current host, while if you are running it on your macOS machine, this should be fine to leave with it, the diff before and after the change would be:

```diff
--- a/meltano_projects/jaffle_superset/meltano.yml
+++ b/meltano_projects/jaffle_superset/meltano.yml
@@ -71,7 +71,7 @@ plugins:
               A list of database driver dependencies can be found here https://superset.apache.org/docs/databases/installing-database-drivers
     config:
       database_name: my_postgres
-      sqlalchemy_uri: postgresql+psycopg2://${PG_USERNAME}:${PG_PASSWORD}@host.docker.internal:${PG_PORT}/${PG_DATABASE}
+      sqlalchemy_uri: postgresql+psycopg2://${PG_USERNAME}:${PG_PASSWORD}@10.1.1.168:${PG_PORT}/${PG_DATABASE}
       tables:
       - model.my_meltano_project.customers
       - model.my_meltano_project.orders
```

- Add Postgres credential to `.env` file:

```bash
echo PG_USERNAME=lineage_ref >> .env
echo PG_PASSWORD=lineage_ref >> .env
```

- Install the Meltano project, run ETL pipeline

```bash
meltano install
meltano run tap-csv target-postgres dbt:run
```

- Start Superset, please note that the `ui` is not a meltano command but a user-defined action in the configuration file.

```bash
meltano invoke superset:ui
```

- In another terminal, run the defined command `load_datasources`

```
meltano invoke superset:load_datasources
```

- Access Superset in a web browser via http://localhost:8088/

We should now see Superset Web Interface:

![](https://user-images.githubusercontent.com/1651790/168570300-186b56a5-58e8-4ff1-bc06-89fd77d74166.png)

#### Create a Dashboard!

Let's try to create a Dashboard on the ETL data in Postgres defined in this Meltano project:

- Click `+ DASHBOARD` , fill a dashboard name, then click `SAVE`, then clieck `+ CREATE A NEW CHART`

![](https://user-images.githubusercontent.com/1651790/168570363-c6b4f929-2aad-4f03-8e3e-b1b61f560ce5.png)

- In new chart view, we should select a chart type and DATASET. Here, I selected `orders` table as the data source and `Pie Chart` chart type:

![](https://user-images.githubusercontent.com/1651790/168570927-9559a2a1-fed7-43be-9f6a-f6fb3c263830.png)

- After clicking `CREATE NEW CHART`, we are in the chart defination view, where, I selected `Query` of `status` as `DIMENSIONS`, and `COUNT(amount)` as `METRIC`. Thus, we could see a Pie Chart per order status's distribution.

![](https://user-images.githubusercontent.com/1651790/168571130-a65ba88e-1ebe-4699-8783-08e5ecf54a0c.png)

- Click `SAVE` , it will ask which dashboard this chart should be added to, after it's selected, click `SAVE & GO TO DASHBOARD`.

![](https://user-images.githubusercontent.com/1651790/168571301-8ae69983-eda8-4e75-99cf-6904f583fc7c.png)

- Then, in the dashboard, we coulds see all charts there. You could see that I added another chart showing customer order count distribution, too:

![](https://user-images.githubusercontent.com/1651790/168571878-30a77057-1f66-448a-9bbd-0dedcee24cc9.png)

- We could set the refresh inteval, or download the dashboard as you wish by clicking the `···` button.

![](https://user-images.githubusercontent.com/1651790/168573874-b5d57919-2866-4b3c-a4e5-55b6e6ef342e.png)

It's quite cool, ah? For now, we have a simple but typical data stack like any hobby data lab with everything open-source!

Imagine we have 100 datasets in CSV, 200 tables in Data warehouse and a couple of data engineers running different projects that consume, generate different application, dashboard, and databases. When someone would like to discovery some of those table, dataset, dashboard and pipelines running across them, and then even modify some of them, it's proven to be quite costly in both communicationand engineering.

Here comes the main part of our reference project: Metadata Discovery.

### Metadata Discovery

Then, we are stepping to deploy the Amundsen with Nebula Graph and Elasticsearch.

> Note: For the time being, the [PR Nebula Graph as the Amundsen backend](https://github.com/amundsen-io/amundsen/pull/1817) is not yet merged.

With Amundsen, we could have all metadata of the whole data stack being discovered and managed in one place. And there are mainly two parts of Amundsen:

- Metadata Ingestion
  - [Amundsen Data builder](https://www.amundsen.io/amundsen/databuilder/)
- Metadata Catalog
  - [Amundsen Frontend service](https://www.amundsen.io/amundsen/frontend/)
  - [Amundsen Metadata service](https://www.amundsen.io/amundsen/metadata/)
  - [Amundsen Search service](https://www.amundsen.io/amundsen/search/)

We will be leveraging `Data builder` to pull metadata from different sources, and persist metadata into the backend storage of the `Meta service` and the backend storage of the `Search service`, then we could search, discover and manage them from the `Frontend service` or through the API of the `Metadata service`.

#### Deploy Amundsen

##### Metadata service

We are going to deploy a cluster of Amundsen with its docker-compose file. As the Nebula Graph backend support is not yet merged, we are referring to my fork.

First, let's clone the repo with all submodules:

```bash
git clone -b amundsen_nebula_graph --recursive git@github.com:wey-gu/amundsen.git
cd amundsen
```

Then, start all catalog services and their backend storage:

```bash
docker-compose -f docker-amundsen-nebula.yml up
```

> You could add `-d` to put the containers running in the background:
>
> ```bash
> docker-compose -f docker-amundsen-nebula.yml up -d
> ```
>
> And this will stop the cluster:
>
> ```bash
> docker-compose -f docker-amundsen-nebula.yml stop
> ```
>
> This will remove the cluster:
>
> ```bash
> docker-compose -f docker-amundsen-nebula.yml down
> ```

Due to this docker-compose file is for developers to play and hack Amundsen easily rather than for production deployment, it's building images from the codebase, which, will take some time for the very first time.

After it's being deployed, please hold on a second before we load some dummy data into its storage with Data builder.

##### Data builder

Amundsen Data builder is just like a Meltano but for ETL of Metadata to `Metadata service` and `Search service`‘s backend storage: Nebula Graph and Elasticsearch. The Data builder here is only a python module and the ETL job could be either run as a script or orchestrated with a DAG platform like Apache Airflow.

With [Amundsen Data builder](https://github.com/amundsen-io/amundsen/tree/main/databuilder) being installed:

```bash
cd databuilder
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install wheel
python3 -m pip install -r requirements.txt
python3 setup.py install
```

Let's call this sample Data builder ETL script to have some dummy data filled in.

```bash
python3 example/scripts/sample_data_loader_nebula.py
```

##### Verify Amundsen

Before accessing Amundsen, we need to create a test user:

```bash
# run a container with curl attached to amundsenfrontend
docker run -it --rm --net container:amundsenfrontend nicolaka/netshoot

# Create a user with id test_user_id
curl -X PUT -v http://amundsenmetadata:5002/user \
    -H "Content-Type: application/json" \
    --data \
    '{"user_id":"test_user_id","first_name":"test","last_name":"user", "email":"test_user_id@mail.com"}'

exit
```

Then we could view UI at [`http://localhost:5000`](http://localhost:5000/) and try to search `test`, it should return some results.

![](https://github.com/amundsen-io/amundsen/raw/master/docs/img/search-page.png)

Then you could click and explore those dummy metadata loaded to Amundsen during the `sample_data_loader_nebula.py` on your own.

Additionally, you could access the Graph Database with Nebula Studio(http://localhost:7001).

> Note in Nebula Studio, the default fields to log in will be:
>
> - Hosts: `graphd:9669`
> - User: `root`
> - Password: `nebula`

This diagram shows some more details on the components of Amundsen:

```asciiarmor
       ┌────────────────────────┐ ┌────────────────────────────────────────┐
       │ Frontend:5000          │ │ Metadata Sources                       │
       ├────────────────────────┤ │ ┌────────┐ ┌─────────┐ ┌─────────────┐ │
       │ Metaservice:5001       │ │ │        │ │         │ │             │ │
       │ ┌──────────────┐       │ │ │ Foo DB │ │ Bar App │ │ X Dashboard │ │
  ┌────┼─┤ Nebula Proxy │       │ │ │        │ │         │ │             │ │
  │    │ └──────────────┘       │ │ │        │ │         │ │             │ │
  │    ├────────────────────────┤ │ └────────┘ └─────┬───┘ └─────────────┘ │
┌─┼────┤ Search searvice:5002   │ │                  │                     │
│ │    └────────────────────────┘ └──────────────────┼─────────────────────┘
│ │    ┌─────────────────────────────────────────────┼───────────────────────┐
│ │    │                                             │                       │
│ │    │ Databuilder     ┌───────────────────────────┘                       │
│ │    │                 │                                                   │
│ │    │ ┌───────────────▼────────────────┐ ┌──────────────────────────────┐ │
│ │ ┌──┼─► Extractor of Sources           ├─► nebula_search_data_extractor │ │
│ │ │  │ └───────────────┬────────────────┘ └──────────────┬───────────────┘ │
│ │ │  │ ┌───────────────▼────────────────┐ ┌──────────────▼───────────────┐ │
│ │ │  │ │ Loader filesystem_csv_nebula   │ │ Loader Elastic FS loader     │ │
│ │ │  │ └───────────────┬────────────────┘ └──────────────┬───────────────┘ │
│ │ │  │ ┌───────────────▼────────────────┐ ┌──────────────▼───────────────┐ │
│ │ │  │ │ Publisher nebula_csv_publisher │ │ Publisher Elasticsearch      │ │
│ │ │  │ └───────────────┬────────────────┘ └──────────────┬───────────────┘ │
│ │ │  └─────────────────┼─────────────────────────────────┼─────────────────┘
│ │ └────────────────┐   │                                 │
│ │    ┌─────────────┼───►─────────────────────────┐ ┌─────▼─────┐
│ │    │ Nebula Graph│   │                         │ │           │
│ └────┼─────┬───────┴───┼───────────┐     ┌─────┐ │ │           │
│      │     │           │           │     │MetaD│ │ │           │
│      │ ┌───▼──┐    ┌───▼──┐    ┌───▼──┐  └─────┘ │ │           │
│ ┌────┼─►GraphD│    │GraphD│    │GraphD│          │ │           │
│ │    │ └──────┘    └──────┘    └──────┘  ┌─────┐ │ │           │
│ │    │ :9669                             │MetaD│ │ │  Elastic  │
│ │    │ ┌────────┐ ┌────────┐ ┌────────┐  └─────┘ │ │  Search   │
│ │    │ │        │ │        │ │        │          │ │  Cluster  │
│ │    │ │StorageD│ │StorageD│ │StorageD│  ┌─────┐ │ │  :9200    │
│ │    │ │        │ │        │ │        │  │MetaD│ │ │           │
│ │    │ └────────┘ └────────┘ └────────┘  └─────┘ │ │           │
│ │    ├───────────────────────────────────────────┤ │           │
│ └────┤ Nebula Studio:7001                        │ │           │
│      └───────────────────────────────────────────┘ └─────▲─────┘
└──────────────────────────────────────────────────────────┘
```





## Connecting the dots, Metadata Discovery

With the basic environment being set up, let's put everything together.

Remember we had ELT some data to PostgreSQL as this?

![](https://user-images.githubusercontent.com/1651790/167540494-01e3dbd2-6ab1-41d2-998e-3b79f755bdc7.png)

How could we let Amundsen discover metadata regarding those data and ETL?

### Extracting Postgres metadata

We started on the data source: Postgres, first.

We install the Postgres Client for python3:

```bash
sudo apt-get install libpq-dev
pip3 install Psycopg2
```

#### Execution of Postgres metadata ETL

Run a script to parse Postgres Metadata:

```bash
export CREDENTIALS_POSTGRES_USER=lineage_ref
export CREDENTIALS_POSTGRES_PASSWORD=lineage_ref
export CREDENTIALS_POSTGRES_DATABASE=warehouse

python3 example/scripts/sample_postgres_loader_nebula.py
```

If you look into the code of the sample script for loading Postgres metadata to Nebula, the main lines are quite straightforward:

```python
# part 1: PostgressMetadata --> CSV --> Nebula Graph
job = DefaultJob(
      conf=job_config,
      task=DefaultTask(
          extractor=PostgresMetadataExtractor(),
          loader=FsNebulaCSVLoader()),
      publisher=NebulaCsvPublisher())

...
# part 2: Metadata stored in NebulaGraph --> Elasticsearch
extractor = NebulaSearchDataExtractor()
task = SearchMetadatatoElasticasearchTask(extractor=extractor)

job = DefaultJob(conf=job_config, task=task)
```

The first job was to load data in path:`PostgressMetadata --> CSV --> Nebula Graph`

- `PostgresMetadataExtractor` was used to extract/pull metadata from Postgres, refer [here](https://www.amundsen.io/amundsen/databuilder/#postgresmetadataextractor) for its documentation.
- `FsNebulaCSVLoader` was used to put extracted data intermediately as CSV files
- `NebulaCsvPublisher` was used to publish metadata in form of CSV to Nebula Graph

The second job was to load in the path: `Metadata stored in NebulaGraph --> Elasticsearch` 

- `NebulaSearchDataExtractor` was used to fetch metadata stored in Nebula Graph
- `SearchMetadatatoElasticasearchTask` was used to make metadata indexed with Elasticsearch.

> Note, in production, we could trigger those jobs either in scripts or with an orchestration platform like Apache Airflow.

#### Verify the Postgres Extraction

Search `payments` or directly visit http://localhost:5000/table_detail/warehouse/postgres/public/payments, you could see the metadata from our Postgres like:

![](https://user-images.githubusercontent.com/1651790/168475180-ebfaa188-268c-4fbe-a614-135d56d07e5d.png)

Then, metadata management actions like adding tags, owners, and descriptions could be done easily as it was in the above screen capture, too.

### Extracting dbt metadata

Actually, we could also pull metadata from [dbt](https://www.getdbt.com/) itself.

The Amundsen [DbtExtractor](https://www.amundsen.io/amundsen/databuilder/#dbtextractor), will parse the `catalog.json` or `manifest.json` file to load metadata to Amundsen storage(Nebula Graph and Elasticsearch).

In above meltano chapter, we had already generated that file with `meltano invoke dbt docs generate`, and the output like the following is telling us the `catalog.json` file:

```log
14:23:15  Done.
14:23:15  Building catalog
14:23:15  Catalog written to /home/ubuntu/ref-data-lineage/meltano_example_implementations/meltano_projects/singer_dbt_jaffle/.meltano/transformers/dbt/target/catalog.json
```

#### Execution of dbt metadata ETL

There is an example script with a sample dbt output files:

The sample dbt files:

```bash
$ ls -l example/sample_data/dbt/
total 184
-rw-rw-r-- 1 w w   5320 May 15 07:17 catalog.json
-rw-rw-r-- 1 w w 177163 May 15 07:17 manifest.json
```

We could load this sample dbt manifest with:

```bash
python3 example/scripts/sample_dbt_loader_nebula.py
```

From this lines of python code, we could tell those process as:

```python
# part 1: Dbt manifest --> CSV --> Nebula Graph
job = DefaultJob(
      conf=job_config,
      task=DefaultTask(
          extractor=DbtExtractor(),
          loader=FsNebulaCSVLoader()),
      publisher=NebulaCsvPublisher())

...
# part 2: Metadata stored in NebulaGraph --> Elasticsearch
extractor = NebulaSearchDataExtractor()
task = SearchMetadatatoElasticasearchTask(extractor=extractor)

job = DefaultJob(conf=job_config, task=task)
```

And the only differences from the Postgres meta ETL is the `extractor=DbtExtractor()`, where it comes with following confiugrations to get below information regarding dbt projects:

- databases_name
- catalog_json
- manifest_json

```python
job_config = ConfigFactory.from_dict({
  'extractor.dbt.database_name': database_name,
  'extractor.dbt.catalog_json': catalog_file_loc,  # File
  'extractor.dbt.manifest_json': json.dumps(manifest_data),  # JSON Dumped objecy
  'extractor.dbt.source_url': source_url})
```

#### Verify the dbt Extraction

Search `dbt_demo` or visit http://localhost:5000/table_detail/dbt_demo/snowflake/public/raw_inventory_value to see:

![](https://user-images.githubusercontent.com/1651790/168479864-2f73ea73-265f-4cd2-999f-e7effbaf3ec1.png)

> Tips: we could optionally enable debug logging to see what had been sent to Elasticsearch and Nebula Graph!
>
> ```diff
> - logging.basicConfig(level=logging.INFO)
> + logging.basicConfig(level=logging.DEBUG)
> ```

Or, alternatively, explore the imported data in Nebula Studio: 

First, click "Start with Vertices", fill in the vertex id: `snowflake://dbt_demo.public/fact_warehouse_inventory`

![](https://user-images.githubusercontent.com/1651790/168480047-26c28cde-5df8-40af-8da4-6ab0203094e2.png)

Then, we could see the vertex being shown as the pink dot. Let's modify the `Expand` options with:

- Direction: Bidirect
- Steps: Single with 3

![](https://user-images.githubusercontent.com/1651790/168480101-7b7b5824-06d9-4155-87c9-798db0dc7612.png)

And double click the vertex(dot), it will expand 3 steps in bidirection:

![](https://user-images.githubusercontent.com/1651790/168480280-1dc88d1b-1f1e-48fd-9997-972965522ef5.png)

From this graph view, the insight of the metadata is extremely easy to be explored, right?

> Tips, you may like to click the 👁 icon to select some properties to be shown, which was done by me before capturing the screen as above.

And, what we had seen in the Nebula Studio echoes the data model of Amundsen metadata service, too:

![](https://www.amundsen.io/amundsen/img/graph_model.png)

Finally, remember we had leveraged dbt to transform some data in meltano, and the menifest file path is `.meltano/transformers/dbt/target/catalog.json`, you can try create a databuilder job to import it.

### Extracting Superset metadata

[Dashboards](https://www.amundsen.io/amundsen/databuilder/databuilder/extractor/dashboard/apache_superset/apache_superset_metadata_extractor.py), [Charts](https://www.amundsen.io/amundsen/databuilder/databuilder/extractor/dashboard/apache_superset/apache_superset_chart_extractor.py) and the [relationships with Tables](https://www.amundsen.io/amundsen/databuilder/databuilder/extractor/dashboard/apache_superset/apache_superset_table_extractor.py) can be extracted by Amundsen data builder, as we already setup a Superset Dashboard, let's try ingesting its metadata.

#### Execution of Superset metadata ETL

The sample superset script will fetch data from Superset and load metadata into Nebula Graph and Elasticsearch.

```python
python3 sample_superset_data_loader_nebula.py
```

If we set the logging level to `DEBUG`, we could actually see lines like:

```python
# fetching metadata from superset
DEBUG:urllib3.connectionpool:http://localhost:8088 "POST /api/v1/security/login HTTP/1.1" 200 280
INFO:databuilder.task.task:Running a task
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): localhost:8088
DEBUG:urllib3.connectionpool:http://localhost:8088 "GET /api/v1/dashboard?q=(page_size:20,page:0,order_direction:desc) HTTP/1.1" 308 374
DEBUG:urllib3.connectionpool:http://localhost:8088 "GET /api/v1/dashboard/?q=(page_size:20,page:0,order_direction:desc) HTTP/1.1" 200 1058
...

# insert Dashboard

DEBUG:databuilder.publisher.nebula_csv_publisher:Query: INSERT VERTEX `Dashboard` (`dashboard_url`, `name`, published_tag, publisher_last_updated_epoch_ms) VALUES  "superset_dashboard://my_cluster.1/3":("http://localhost:8088/superset/dashboard/3/","my_dashboard","unique_tag",timestamp());
...

# insert a DASHBOARD_WITH_TABLE relationship/edge

INFO:databuilder.publisher.nebula_csv_publisher:Importing data in edge files: ['/tmp/amundsen/dashboard/relationships/Dashboard_Table_DASHBOARD_WITH_TABLE.csv']
DEBUG:databuilder.publisher.nebula_csv_publisher:Query:
INSERT edge `DASHBOARD_WITH_TABLE` (`END_LABEL`, `START_LABEL`, published_tag, publisher_last_updated_epoch_ms) VALUES "superset_dashboard://my_cluster.1/3"->"postgresql+psycopg2://my_cluster.warehouse/orders":("Table","Dashboard","unique_tag", timestamp()), "superset_dashboard://my_cluster.1/3"->"postgresql+psycopg2://my_cluster.warehouse/customers":("Table","Dashboard","unique_tag", timestamp());
```

#### Verify the Superset Dashboard Extraction

By searching it in Amundsen, we could the Dashboard info now. And we could verify it from Nebula Studio, too.

![](https://user-images.githubusercontent.com/1651790/168719624-738323dd-4c6e-475f-a370-f149181c6184.png)

> Note, see also the Dashboard's model in Amundsen from [the dashboard ingestion guide](https://www.amundsen.io/amundsen/databuilder/docs/dashboard_ingestion_guide/):
>
> ![dashboard_graph_modeling](https://www.amundsen.io/amundsen/databuilder/docs/assets/dashboard_graph_modeling.png?raw=true)

### Preview data with Superset

Superset could be used to preview Table Data like this. Corresponding documentation could be referred [here](https://www.amundsen.io/amundsen/frontend/docs/configuration/#preview-client), where the API of `/superset/sql_json/` will be called by Amundsen Frontend.

![](https://github.com/amundsen-io/amundsenfrontendlibrary/blob/master/docs/img/data_preview.png?raw=true)

### Enable Data lineage

By default, data lineage was not enabled, we could enable it by:

0. Go to the Amundsen repo, that's also where we run the `docker-compose -f docker-amundsen-nebula.yml up` command

```bash
cd amundsen
```

1. Modify frontend  JS configuration:

```diff
--- a/frontend/amundsen_application/static/js/config/config-default.ts
+++ b/frontend/amundsen_application/static/js/config/config-default.ts
   tableLineage: {
-    inAppListEnabled: false,
-    inAppPageEnabled: false,
+    inAppListEnabled: true,
+    inAppPageEnabled: true,
     externalEnabled: false,
     iconPath: 'PATH_TO_ICON',
     isBeta: false,
```

2. Now let's run again build for docker image, where the frontend image will be rebuilt.

```bash
docker-compose -f docker-amundsen-nebula.yml build
```

Then, rerun the `up -d` to ensure frontend container to be recreated with new configuration:

```bash
docker-compose -f docker-amundsen-nebula.yml up -d
```

We could see something like this:

```bash
$ docker-compose -f docker-amundsen-nebula.yml up -d
...
Recreating amundsenfrontend           ... done
```

After that, we could visit http://localhost:5000/lineage/table/gold/hive/test_schema/test_table1 to see the `Lineage` is shown as:

![](https://user-images.githubusercontent.com/1651790/168838731-79d0e3bc-439e-4f6b-8ef7-83b37e9bcb12.png)

We could click `Downstream`(if there is) to see downstream resources of this table:

![](https://user-images.githubusercontent.com/1651790/168839251-efd523af-d729-44cf-a40b-fa83a0852654.png)

Or click Lineage to see the graph:

![](https://user-images.githubusercontent.com/1651790/168838814-e6ff5152-c24b-470e-a46a-48f183ba7201.png)

There are API for lineage query, too. Here is an example to query that with cURL, where we leverage the netshoot container as we did before for user creation.

```bash
docker run -it --rm --net container:amundsenfrontend nicolaka/netshoot

curl "http://amundsenmetadata:5002/table/snowflake://dbt_demo.public/raw_inventory_value/lineage?depth=3&direction=both"
```

The above API call was to query linage on both upstream and downstream direction, with depth 3 for table `snowflake://dbt_demo.public/raw_inventory_value`.

And the result should be like:

```json
{
    "depth": 3,
    "downstream_entities": [
        {
            "level": 2,
            "usage": 0,
            "key": "snowflake://dbt_demo.public/fact_daily_expenses",
            "parent": "snowflake://dbt_demo.public/fact_warehouse_inventory",
            "badges": [],
            "source": "snowflake"
        },
        {
            "level": 1,
            "usage": 0,
            "key": "snowflake://dbt_demo.public/fact_warehouse_inventory",
            "parent": "snowflake://dbt_demo.public/raw_inventory_value",
            "badges": [],
            "source": "snowflake"
        }
    ],
    "key": "snowflake://dbt_demo.public/raw_inventory_value",
    "direction": "both",
    "upstream_entities": []
}
```

In fact, this lineage data was just extracted and loaded during our [DbtExtractor](https://github.com/amundsen-io/amundsen/blob/main/databuilder/databuilder/extractor/dbt_extractor.py) execution, where `extractor.dbt.{DbtExtractor.EXTRACT_LINEAGE}` by default was `True`, thus lineage metadata were created and loaded to Amundsen.

#### Get lineage in Nebula Graph

Two of the advantages to use a Graph Database as Metadata Storage are:

- The graph query itself is a flexible DSL for lineage API, for example, this query helps us do the equivalent query of the Amundsen metadata API for fetching lineage:

```cypher
MATCH p=(t:Table) -[:HAS_UPSTREAM|:HAS_DOWNSTREAM *1..3]->(x)
WHERE id(t) == "snowflake://dbt_demo.public/raw_inventory_value" RETURN p
```

- We could now even query it in Nebula Graph Studio's console, and click `View Subgraphs` to make it rendered in a graph view then.

![](https://user-images.githubusercontent.com/1651790/168844882-ca3d0587-7946-4e17-8264-9dc973a44673.png)

![](https://user-images.githubusercontent.com/1651790/168845155-b0e7a5ce-3ddf-4cc9-89a3-aaf1bbb0f5ec.png)

#### Extract Data Lineage

##### Dbt

As mentioned above, [DbtExtractor](https://www.amundsen.io/amundsen/databuilder/#dbtextractor) will extract table level lineage, together with other information defined in the dbt ETL pipeline.

##### Open Lineage

The other linage extractor out-of-the-box in Amundsen is [OpenLineageTableLineageExtractor](https://www.amundsen.io/amundsen/databuilder/#openlineagetablelineageextractor).

[Open Lineage](https://openlineage.io/) is an open framework to collect lineage data from different sources in one place, which can output linage information as JSON files to be extracted by [OpenLineageTableLineageExtractor](https://www.amundsen.io/amundsen/databuilder/#openlineagetablelineageextractor):

```python
dict_config = {
    # ...
    f'extractor.openlineage_tablelineage.{OpenLineageTableLineageExtractor.CLUSTER_NAME}': 'datalab',
    f'extractor.openlineage_tablelineage.{OpenLineageTableLineageExtractor.OL_DATASET_NAMESPACE_OVERRIDE}': 'hive_table',
    f'extractor.openlineage_tablelineage.{OpenLineageTableLineageExtractor.TABLE_LINEAGE_FILE_LOCATION}': 'input_dir/openlineage_nd.json',
}
...

task = DefaultTask(
    extractor=OpenLineageTableLineageExtractor(),
    loader=FsNebulaCSVLoader())
```

### Recap, bird's-eye view of Metadata Discovery

The whole idea of Metadata Discovery is to:

- Put all components in the stack as Metadata Sources(from any DB or DW to dbt, Airflow, Openlineage, Superset, etc.)
- Run metadata ETL with Databuilder(as a script, or DAG) to store and index with Nebula Graph(or other Graph Database) and Elasticsearch
- Consume, manage, and discover metadata from Frontend UI(with Superset for preview) or API
- Have more possibilities, flexibility, and insights on Nebula Graph from queries and UI

![](https://user-images.githubusercontent.com/1651790/168849779-4826f50e-ff87-4e78-b17f-076f91182c43.svg)

## Upstream Projects

All projects used in this reference project are listed below in lexicographic order.

- Amundsen
- Apache Airflow
- Apache Superset
- dbt
- Elasticsearch
- meltano
- Nebula Graph
- Open Lineage
- singer

