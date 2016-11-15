# The install guides have been written as Jupyter Notebooks

To use them, you must start a jupyter server.

To install python on a ubuntu system, do

    sudo apt-get install -y python virtualenv virtualenvwrapper
    
    sudo yum install python-virtualenv
    
Then you must set up virtualenv
Add this line to your ~/.bashrc or ~/.bash_profile or ~/.profile

    export WORKON_HOME=$HOME/.virtualenvs
    mkdir -p $WORKON_HOME
    source  /usr/share/virtualenvwrapper/virtualenvwrapper.sh

Start a new terminal (i.e. one that have the updated environment you just added)
    
Create a new virtual python environment (jupyter-digitv-releasetest is just a name I chose, you can pick another)

    mkvirtualenv jupyterEnvironment -p /usr/bin/python3
    pip install -U pip
    
To switch to this new environment, use (mkvirtualenv switches you automatically)
    
    workon jupyterEnvironment
    
To install jupyter in your new virtual python environment, do

    pip install jupyter

We use a bash kernel for jupyter for the tests, so this kernel must also be installed

    pip install -U flit
    flit installfrom https://github.com/takluyver/bash_kernel/archive/master.zip
    python -m bash_kernel.install
    
    
Install the jupyter extensions, if you want to be able to see table of contents and the like
    
    pip install jupyter_contrib_nbextensions
    jupyter contrib nbextension install --sys-prefix --symlink
    
Enable the TableOfContents extension

    jupyter nbextension enable toc2/main
    
Then start jupyter and open the install guide

    jupyter notebook --ip=$(hostname) --port=28888

To see what other extensions there are and configure them, go to
    
    http://localhost:8888/nbextensions