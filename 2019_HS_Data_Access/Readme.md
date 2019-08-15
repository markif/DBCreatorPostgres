# Install Jupyter Notebook

There are several options to install Jupyter Notebook. 

One option is to install Anaconda which comes with Jupyter Notebook (please follow [this](
https://jupyter.readthedocs.io/en/latest/install.html#id3) procedure). The downside of this option is that you have to manage the necessary dependencies (e.g. postgresql drivers) yourself.

Alternatively, you can use Docker to run Jupyter Notebooks (the Docker Image brings alonge the necessary dependencies). 

## Install Docker

See https://docs.docker.com/install

## Run Jupyter Notebook

Run the Jupyter Notebook with disabled authentication (not a recommended practice) as a background process.

```bash
# set the path to your Jupyter Notebook files
JUPYTER_FILES=${PWD}
# On Windows Powershell please use
# $JUPYTER_FILES=${PWD}
docker run --name datascience-notebook -p 8888:8888 -v ${JUPYTER_FILES}:/home/jovyan/work -d i4ds/datascience-notebook start-notebook.sh --NotebookApp.token=''
# start your browser
firefox http://127.0.0.1:8888
```

Run the Jupyter Notebook with authentication (see console output for the url to use in your browser).

```bash
# set the path to your Jupyter Notebook files
JUPYTER_FILES=${PWD}
# On Windows Powershell please use
# $JUPYTER_FILES=${PWD}
docker run --name datascience-notebook -p 8888:8888 -v ${JUPYTER_FILES}:/home/jovyan/work -it --rm i4ds/datascience-notebook
```

Run a Jupyter Hub with authentication (see console output for the url to use in your browser).

```bash
# set the path to your Jupyter Notebook files
JUPYTER_FILES=${PWD}
# On Windows Powershell please use
# $JUPYTER_FILES=${PWD}
docker run --name datascience-notebook -p 8888:8888 -e JUPYTER_ENABLE_LAB=yes -v ${JUPYTER_FILES}:/home/jovyan/work -it --rm i4ds/datascience-notebook
```
