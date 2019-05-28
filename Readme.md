# Install Docker

See https://docs.docker.com/install

# Setup Database

Following steps setup a Postgres database and use I4DS's datascience notebook to create tables and add data.

You might need to cleanup the environment.

```bash
docker stop datascience-notebook
docker rm datascience-notebook

docker stop postgres-db
docker rm postgres-db 

docker network rm dbnet
```

Create a common network used for communication between the Docker containers (see https://nickjanetakis.com/blog/docker-tip-35-connect-to-a-database-running-on-your-docker-host or https://nickjanetakis.com/blog/docker-tip-65-get-your-docker-hosts-ip-address-from-in-a-container after https://github.com/jupyter/docker-stacks/issues/542 got resolved)

```bash
docker network create -d bridge --subnet 192.168.0.0/24 --gateway 192.168.0.1 dbnet
```

Start the Docker container providing the postgres database

```bash
docker run --name postgres-db --net=dbnet -p 5432:5432 -e POSTGRES_DB=bank_db -e POSTGRES_USER=bank_user -e POSTGRES_PASSWORD=bank_pw -d postgres:11.3

# test connection with psql client using another Docker container
docker run -it --net=dbnet --rm postgres:11.3 psql -h 192.168.0.1 -U bank_user -d bank_db
```

Start a Docker container using I4DS's datascience notebook
```bash
# set the path to Import_Data.ipynb
JUPYTER_FILES=$(pwd)/ExplorativeAnalysisChallenge_HS2019/
# start Jupyter Notebook with disabled authentication (not a recommended practice)
docker run --name datascience-notebook --net=dbnet -p 8888:8888 -v "${JUPYTER_FILES}":/home/jovyan/work -d i4ds/datascience-notebook start-notebook.sh --NotebookApp.token=''
```

Execute Import_Data.ipynb Notebook to load the data.

```bash
firefox http://127.0.0.1:8888/notebooks/work/Import_Data.ipynb
```

Or connect to datascience-notebook and execute the script from the console.

```bash
docker exec -it datascience-notebook start.sh
jupyter nbconvert --ExecutePreprocessor.timeout=360 --to notebook --execute ~/work/Import_Data.ipynb
```

Extract the database (in order to share it with students)

```bash
rm -f ${JUPYTER_FILES}/dbdump.sqlc
docker run -it --net=dbnet -v "${JUPYTER_FILES}":/dump --rm postgres:11.3 pg_dump --format=c --file=/dump/dbdump.sqlc -h 192.168.0.1 -U bank_user -d bank_db
```

Copy the extracted database to the corresponding git repository where it will be made public to the students

```bash
cp ${JUPYTER_FILES}/dbdump.sqlc ../$(basename ${JUPYTER_FILES})
```



