
<!-- omit in toc -->
Table Of Contents
=================



Setup Spark Standalone
======================

Download Spark
--------------

for instance untar spark under `~/.local/lib/` and make a symlink
```bash
ln -s ~/.local/lib/spark-YOUR-SPECIFIC-VERSION ~/.local/lib/spark
```

add the following to your .bashrc by running
```bash
# Note the quotes arround 'EOF' - they won't replace system variables $HOME etc.
cat >> .bashrc << 'EOF'
export SPARK_HOME=$HOME/.local/lib/spark
export PATH=$SPARK_HOME/bin:$SPARK_HOME/sbin:$PATH
# we default to using python3 as pyspark python
export PYSPARK_PYTHON=python3
EOF
```

Setup Custom Spark Configuration
--------------------------------

to make your own writable copy of spark's default configurations execute the following in the remote terminal
```bash
mkdir ~/conf
cd ~/conf
cp $SPARK_HOME/conf/spark-defaults.conf spark.conf
cp $SPARK_HOME/conf/log4j.properties log4j-driver.properties
cp log4j-driver.properties log4j-executor.properties
```

add the following configurations to the spark.conf by executing
```bash
# Note the missing quotes arround the first EOF - this means all system variables $HOME etc. will be replaced directly
cat >> ~/conf/spark.conf << EOF
spark.files                         $HOME/conf/log4j-executor.properties
spark.driver.extraJavaOptions       -Dlog4j.configuration=file:$HOME/conf/log4j-driver.properties
spark.executor.extraJavaOptions     -Dlog4j.configuration=log4j-executor.properties
```

IMPORTANT:
- Finally, these have to be merged, such that there is always only one key of `spaprk.files`/`spark.driver.extraJavaOptions`/`spark.executor.extraJavaOptions` in the overall `spark.conf` file.
- Spark properties parameters exist only once, if a parameter is defined twice, it is just overwritten. 
- you can do this easily using JupyterLab or RStudio from the lab.



Configuring Cluster Size using $SPARK_HOME/conf/spark-env.sh
------------------------------------------------------------

e.g. execute the following (only once!) so that when running ``start-slave.sh ...``
actually two workers are started with 2 CPU cores and 2GB memory each

```bash
cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh
# Note the quotes arround 'EOF' - they won't replace system variables $HOME etc.
cat >> $SPARK_HOME/conf/spark-env.sh <<'EOF'
SPARK_WORKER_CORES=2
SPARK_WORKER_INSTANCES=2
SPARK_WORKER_MEMORY=2g
EOF
```

Start/Stop Cluster
------------------

on linux shell execute
```bash
start-master.sh
```
as soon as the master runs, go to http://localhost:8080 by default and copy the `CLUSTERURL`, then
```bash
start-slave.sh CLUSTERURL
```

stop everything by running the counterpart scripts in reversed order
```bash
stop-slave.sh
stop-master.sh
```

As the `CLUSTERURL` will likely stay for your system, you can add aliases to your shell configuration file
```bash
# Note the quotes arround 'EOF' - they won't replace system variables $HOME etc.
cat >> ~/.bashrc << 'EOF'
export SPARK_TEST_CLUSTER_URL=">>>>>>>>>>>>> ENTER YOUR SPECIFIC CLUSTERURL HERE <<<<<<<<<<<<<<<<<<"
alias start-spark-test-cluster="start-master.sh && start-slave.sh $SPARK_TEST_CLUSTER_URL"
alias stop-spark-test-cluster="stop-slave.sh && stop-master.sh"
EOF
source ~/.bashrc
```


and also add the `$SPARK_TEST_CLUSTER_URL` to your spark-config
```bash
# Note the missing quotes arround the first EOF - this means all system variables $HOME etc. will be replaced directly
cat >> ~/conf/spark.conf << EOF
spark.master        $SPARK_TEST_CLUSTER_URL
```
IMPORTANT
- if multiple `spark.master` config entries exist, decide for one!



--------------------


You may rightly noticed that there are `start-all.sh`/`stop-all.sh` scripts which seem to be for exactly this purpose. However they are intended more for a real standalone cluster with multiple machines and concretely configure everything with ssh. Hence you would need to setup password less ssh-access to your own ubuntu, which is probably a bit overcomplicated.



Connecting to the Cluster
-------------------------

from Windows Linux Subsystem (WSL) just use `SPARK_TEST_CLUSTER_URL` as your `--master`, e.g.
```bash
pyspark --master $SPARK_TEST_CLUSTER_URL
```
or
```bash
spark-shell --master $SPARK_TEST_CLUSTER_URL
```
will connect to the standalone cluster now


References
----------

build upon the following two resources
- https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-standalone-example-2-workers-on-1-node-cluster.html
- https://spark.apache.org/docs/latest/spark-standalone.html



JupyterLab on WSL
=================

Jupyter is well supported by spark, however needs to be started manually from commandline to set the `properties-file`
The following steps will guide you through setting up `pyspark` with `jupyter-lab`.


Jupyter Installation
--------------------

execute on remote lab commandline
```bash
python3 -m pip install --upgrade pip --user
python3 -m pip install jupyter --user
python3 -m pip install jupyterlab --user
```

we need to add the local bin folder to our Path and additionally can also add an alias to your .bashrc file for some custom jupyterlab options
```bash
# Note the quotes arround 'EOF' - they won't replace system variables $HOME etc.
cat >> ~/.bashrc << 'EOF'
export PATH=$HOME/.local/bin:$PATH
alias myjupyterlab="jupyter lab --no-browser --port=8888 --NotebookApp.token=''"
EOF
```

Jupyter PySpark-Kernel
----------------------

add a pyspark Kernel to jupyter by executing the following on the remote command line

```bash
# we directly create a softlink to have visibility in JupyterLab (hidden files are currently not supported)
ln -s $HOME/.local/share/jupyter/kernels ~/jupyter_kernels
mkdir jupyter_kernels/pyspark/
cp jupyter_kernels/python3/logo* jupyter_kernels/pyspark/

# Note the missing quotes arround the first EOF - this means all system variables $HOME etc. will be replaced directly
cat > jupyter_kernels/pyspark/kernel.json << EOF
{
 "display_name": "PySpark",
 "language": "python",
 "argv": [
  "$HOME/.local/share/jupyter/kernels/pyspark/kernel.sh",
  "-f",
  "{connection_file}"
 ]
}
EOF

# Note the quotes arround 'EOF' - they won't replace system variables $HOME etc.
cat > jupyter_kernels/pyspark/kernel.sh << 'EOF'
#!/usr/bin/env bash
# we use this bash file to have full bash support (instead of jupyter kernel "env" plain text environment variables)
module pyspark

# this mimicks $SPARK_HOME/bin/pyspark configurations
export PYTHONPATH="$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.7-src.zip"
export PYTHONSTARTUP="$SPARK_HOME/python/pyspark/shell.py"
export PYSPARK_PYTHON=/usr/bin/python3

# this is an internal detail, how spark arguments are passed to pyspark
# indeed it is completely setup from Python itself, unlike you might think when working with pyspark
# mind the special last argument "pyspark-shell"

export PYSPARK_SUBMIT_ARGS="--properties-file $HOME/conf/spark.conf --conf spark.repl.class.outputDir=$(mktemp -d ~/tmp/spark_classes_XXXXXXXXXXXX) pyspark-shell"

# this is the critical part, and should be at the end of your script:
exec /usr/bin/python3 -m ipykernel_launcher --profile pyspark $@
EOF

chmod +x $HOME/.local/share/jupyter/kernels/pyspark/kernel.sh
mkdir ~/tmp

# create the extra pyspark profile to have a separate history
ipython profile create pyspark
echo "load_subconfig('ipython_config.py', profile='default')" >> ~/.ipython/profile_pyspark/ipython_config.py
echo "load_subconfig('ipython_kernel_config.py', profile='default')" >> ~/.ipython/profile_pyspark/ipython_kernel_config.py
```

Notes
- the `py4j-0.10.7-src.zip` in `PYTHONPATH` may likely change between spark versions.
- the `pyspark-shell` extra argument at the end of `PYSPARK_SUBMIT_ARGS` is decisive! It communicates with spark-submit main class
- we cannot use `$HOME`/`$USER` or the like in the `kernel.json` itself. Still you see this above, as it will be replaced as soon as executed, printing the concrete home directory in `kernel.json` instead of the abstract reference. See https://github.com/jupyter/notebook/issues/3469 and https://github.com/jupyterhub/jupyterhub/issues/847
- you can add `"OLD_PYTHONSTARTUP"` to as environment variable, which will be executed by `.../pyspark/shell.py` - an implementation detail of `$SPARK_HOME/bin/pyspark`



How to change options in Runtime
--------------------------------

Just adapt the pyspark kernel.sh `~/jupyter_kernels/pyspark/kernel.sh` within the running JupyterLab and restart the kernel

----------------

In general to list the current kernel locations, run
```bash
jupyter kernelspec list
```
and as JuypterLab currently does not support hidden files https://github.com/jupyter/notebook/issues/3812 you may want to open a Terminal with JupyterLab and  create symbolic links like the one above by running
```bash
ln -s $HOME/.local/share/jupyter/kernels ~/jupyter_kernels
```
in order to access hidden directories



References
----------

- best and easiest https://raspi.farm/howtos/configure-spark-for-jupyter/. Essentially you create a jupyter-kernel-spec file analog to this
- similar https://docs.anaconda.com/ae-notebooks/admin-guide/install/config/custom-pyspark-kernel/
- better approach to environment variable handling https://github.com/jupyterhub/jupyterhub/issues/847
- I confirmed the setting by looking into the source code. It indeed is as simple as they suggest
- ipython profile configuration https://ipython.readthedocs.io/en/stable/development/config.html



R on WSL
========

R + Jupyter Kernel on WSL
-------------------------

Note: You may want to use Anaconda instead of system R, a bit more setup but might be preferable.

first install R + binary dependencies
```bash
sudo apt-get install r-base libzmq3-dev libcurl4-openssl-dev libssl-dev libxml2-dev
```

then after starting `R`, install packages via R commandline
```R
install.packages(c('crayon', 'pbdZMQ', 'devtools', 'tidyverse', 'sparklyr'))
devtools::install_github(paste0('IRkernel/', c('repr', 'IRdisplay', 'IRkernel')))
```

and finally setup the R Jupyter Kernel, still in R commandline
```R
IRkernel::installspec()
```


RStudio Server on WSL
---------------------

how to install: https://www.rstudio.com/products/rstudio/download-server/
how to manage: https://support.rstudio.com/hc/en-us/articles/200532327-Managing-the-Server

RStudio Server lives on : http://localhost:8787

add the following (probably adapted) to `~/.Renviron`
```bash
SPARK_HOME=~/.local/lib/spark
JAVA_HOME=/usr/lib/jvm/default-java
```

you can login with your WSL login name+password


References
----------

- IRKernel https://irkernel.github.io/installation/ + google search + missing xml2 package dependencies



Scala Interface on WSL
======================

Probably best is still JupyterLab... maybe Apache Toree, however currently the scala landscape still looks difficult... with special shell aml and toree



Python/R from Windows itself
============================

unfortunately nothing worked so far.... connection problems
