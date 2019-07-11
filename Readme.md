# Install Docker

See https://docs.docker.com/install

# Create a Database Dump

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
DATABASE="bank_db"
docker run --name postgres-db --net=dbnet -p 5432:5432 -e POSTGRES_USER=db_admin -e POSTGRES_PASSWORD=db_admin_0815_pw -d postgres:11.3

# test connection with psql client using another Docker container
docker run -it --net=dbnet --rm postgres:11.3 psql -h 192.168.0.1 -U db_admin -d ${DATABASE}
```

Start a Docker container using I4DS's datascience notebook
```bash
# set the project name
PROJECT_NAME="2019_HS_Challenge_ExplorativeAnalysis"
# set the path to Import_Data.ipynb
JUPYTER_FILES=$(pwd)/${PROJECT_NAME}

# change rights and copy template file (only if necessary)
chmod -R 777 ${JUPYTER_FILES}
cp Import_Data.ipynb ${JUPYTER_FILES}

# start Jupyter Notebook with disabled authentication (not a recommended practice)
docker run --name datascience-notebook --net=dbnet -p 8888:8888 -v "${JUPYTER_FILES}":/home/jovyan/work -d i4ds/datascience-notebook start-notebook.sh --NotebookApp.token=''
```

Copy the data.zip (with the *.csv files containing the data of the tables) to ${JUPYTER_FILES}

Execute Import_Data.ipynb Notebook to load the data (you will need to adapt the settings).

```bash
firefox http://127.0.0.1:8888/notebooks/work/Import_Data.ipynb
```

Or connect to datascience-notebook and execute the script from the console.

```bash
docker exec -it datascience-notebook start.sh
jupyter nbconvert --ExecutePreprocessor.timeout=360 --to notebook --execute ~/work/Import_Data.ipynb
```

Extract the database and users (e.g. to share it with students)

```bash
# prepare folder structure and files
EXPORT_FOLDER=${JUPYTER_FILES}"/export"
mkdir -p ${EXPORT_FOLDER}
docker run -it --net=dbnet -v "${JUPYTER_FILES}":/dump --rm postgres:11.3 pg_dump --format=c --file=/dump/export/dbdump.sqlc -h 192.168.0.1 -U db_admin -d ${DATABASE}
docker run -it --net=dbnet -v "${JUPYTER_FILES}":/dump --rm postgres:11.3 pg_dumpall -h 192.168.0.1 -U db_admin -g -f "/dump/export/users.sql"
pushd $(pwd) && cd ${JUPYTER_FILES} && tar cfvz export.tar.gz export && popd
rm -rf ${EXPORT_FOLDER}
```

# Import Dump to Database

Download the export.tar.gz file containing the database dump to import.

```bash
# make sure the database is running (here we use a docker container)
docker run --name explorative-analysis-db --net=dbnet -p 5432:5432 -e POSTGRES_USER=db_admin -e POSTGRES_PASSWORD=db_admin_0815_pw -e POSTGRES_DB=${DATABASE} -d postgres:11.3
# connect to the database server and make sure export.tar.gz is available
docker run -it --net=dbnet -v "$(pwd)":/dump --rm postgres:11.3 /bin/bash

# extract the dump
tar -xvzf export.tar.gz
DATABASE="bank_db" # or "warenkorb_db"
# restore user/role
psql -h 192.168.0.1 -U db_admin -d ${DATABASE} -f export/users.sql
# restore the data - the password is bank_pw (see above)
pg_restore -h 192.168.0.1 -U db_admin -d ${DATABASE} --format=c export/dbdump.sqlc
# cleanup
rm -rf export && rm -f export.tar.gz

# test if the data is available
psql -h 192.168.0.1 -U db_user -d ${DATABASE}
SELECT COUNT(*) FROM trans;
exit
```


# On a Database Server

## Install Postgres

```bash
# install postgres
sudo apt install postgresql postgresql-contrib

# create db_admin user
sudo su - postgres
# replace "peer" with "md5" for "local   all             all"
# and add "host  all  all 0.0.0.0/0 md5"
nano /etc/postgresql/10/main/pg_hba.conf 
# add listen_addresses = '*'
nano nano /etc/postgresql/10/main/postgresql.conf

# restart postgres
sudo service postgresql restart
# create db_admin user
createuser -sPE db_admin
createdb db_admin
# check if login works
psql -U db_admin
```

## Import Data

```bash
# copy dump to database server
scp -i switch-cloud.key 2019_HS_Challenge_ExplorativeAnalysis/export.tar.gz ubuntu@86.119.36.94:~/downloads
# login
ssh -i switch-cloud.key ubuntu@86.119.36.94
cd downloads

# extract the dump
tar -xvzf export.tar.gz
DATABASE="bank_db" # or "warenkorb_db"
createdb -U db_admin ${DATABASE}
createdb -U db_admin db_user
# restore user/role
psql -U db_admin -d ${DATABASE} -f export/users.sql
# restore the data - the password is bank_pw (see above)
pg_restore -U db_admin -d ${DATABASE} --format=c export/dbdump.sqlc
# cleanup
rm -rf export && rm -f export.tar.gz
```

