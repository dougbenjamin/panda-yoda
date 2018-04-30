# panda-yoda

## Requirements

- python 2.7+ (haven't actually tested with Python 3)
- mpi4py
- yampl
- python-yampl

##  Installation

If you are using a non-standard MPI library, which is common for most supercomputers, you will need to compile the `mpi4py` module against your local MPI libraries. In order to compile the `mpi4py` module you will need to compile the `Cython` module which `mpi4py` uses to wrap the C++ MPI library functions in Python.

In all cases you will need to compile the `yampl` program and the `python-yampl` wrapper for Yoda to use during communications with AthenaMP.

Here are rough instructions for this process, there are specific instructions for some sites further below.

## Generic Instructions

```bash
# setup environment
# typically I setup the virtualenv of Harvester at this point
# if your HPC system uses modules to setup environments you may
# need to source some modules for an older GNU or move from 
# intel to GNU compilers, etc.

# I'll assume you are in the Harvester virtualenv base directory
# referenced by $VIRTUAL_ENV

# Custom build Cython with local compilers
git clone git@github.com:cython/cython.git
cd cython
git checkout tags/0.25.2 # later tags may also work (haven't tested)
# here cc and CC are place holders for whichever compiler your
# local system requires
CC=cc CXX=CC python setup.py build
export PYTHONPATH=$VIRTUAL_ENV/lib64/python2.7/site-packages
# this installs the python module in the same site-packages area
# as harvester, etc.
CC=cc CXX=CC python setup.py install --prefix=$VIRTUAL_ENV

cd ../

# Custom build mpi4py with local compilers for worker nodes
git clone git@github.com:mpi4py/mpi4py.git
cd mpi4py
git checkout tags/2.0.0 # later tags may work (haven't tested)
##  edit mpi.cfg
##  under [mpi]
##  uncomment mpicc and mpicxx
##  change compiler to point at your local compilers (cc & CC) in this example
CC=cc CXX=CC python setup.py build
CC=cc CXX=CC python setup.py install --prefix=$VIRTUAL_ENV

cd ../

# next you need to install yampl which AthenaMP uses to communicate
git clone git@github.com:vitillo/yampl.git
cd yampl
# on some systems like Theta you need "LDFLAGS=-dynamic" to force dynamic linking
# if you find it causing problems on your system, remove it
./configure --prefix=$VIRTUAL_ENV/install CC=cc CXX=CC LDFLAGS=-dynamic
make -j 10 install
# in my case this fails in the zeromq folder
# so go into the zeromq directory and build by hand
cd zeromq
./configure --prefix=$VIRTUAL_ENV/install CC=cc CXX=CC LDFLAGS=-dynamic
make -j 10 install
cd ..
# continue with yampl build
make -j 10 install

cd ../

# now you need to install the python bindings for yampl
git clone git@github.com:vitillo/python-yampl.git
cd python-yampl
# then run
python setup.py build_ext 
# the last command may fail, if so,
# edit setup.py such that inside the 'include_dirs' inside the 'Extension' have
# both the path to the '$VIRTUAL_ENV/install/include/yampl' 
# and '$VIRTUAL_ENV/install/include' and the 'extra_link_args' has 
# '-L/path/to/yampl/install/lib'. Then run it again
# Run this to install the module
python setup.py install

cd ../

# install yoda into the $VIRTUAL_ENV/python2.7/site-packages
pip install git+git://github.com/PanDAWMS/panda-yoda
```

You should be able to run the `yoda_droid` command from the command line at this point and have it work. There is configuration now needed. See the configuration section for that.

## Uninstall or Update

In the ideal case yoda is updated via this command while inside the virtual environment:
```bash
pip install --update git+git://github.com/PanDAWMS/panda-yoda
```

In my experience with these pip installs from github, this method can fail. If it does you must do the following to update or uninstall:
```bash
rm -rf $VIRTUAL_ENV/lib/python2.7/site-packages/pandayoda
rm -rf $VIRTUAL_ENV/lib/python2.7/site-packages/panda_yoda-0.0.1-py2.7.egg-info
pip install git+git://github.com/PanDAWMS/panda-yoda
```


# Install on NERSC/Edison

Custom installation for Edison at NERSC

```bash
# setup environment
module load python/2.7.9
module load mpi4py/1.3.1
module load virutalenv
module swap PrgEnv-intel PrgEnv-gnu

# create virtualenv area if not already existing
virtualenv panda
cd panda
# upgrade the local version of pip and tell it how to reach outside NERSC
pip install --upgrade --index-url=http://pypi.python.org/simple/ --trusted-host pypi.python.org  pip

pip install git+git://github.com/PanDAWMS/panda-yoda

# next you need to install yampl
git clone git@github.com:vitillo/yampl.git
cd yampl
./configure --prefix=$PWD/install CC=cc CXX=CC LDFLAGS=-dynamic
make -j 10 install
# in my case this fails in the zeromq folder
# so do this
cd zeromq
./configure --prefix=$PWD/install CC=cc CXX=CC LDFLAGS=-dynamic
make -j 10 install
cd ..
make -j 10 install
cd ..

# now you need to install the python bindings for yampl
git clone git@github.com:jtchilders/python-yampl.git
cd python-yampl
# edit setup.py such that inside the 'include_dirs' inside the 'Extension' have
# both the path to the '/path/to/yampl/install/include/yampl' 
# and '/path/to/yampl/install/include' and the 'extra_link_args' has 
# '-L/path/to/yampl/install/lib'. 

# then run
python setup.py build_ext 
# the last command may fail, if so, remove the '-dynamic' from it and run by hand
# then run
python setup.py install


# in order to run you need to make sure LD_LIBRARY_PATH includes
export LD_LIBRARY_PATH=/path/to/yampl/install/lib:$LD_LIBRARY_PATH

```

# Install on ALCF/Theta:
```bash

# create install area
INSTALL_AREA=/path/to/yoda
mkdir $INSTALL_AREA
cd $INSTALL_AREA

# build using GNU not Intel compilers
module swap PrgEnv-intel PrgEnv-gnu

# Custom build Cython with local compilers for Theta worker nodes
git clone git@github.com:cython/cython.git
cd cython
git checkout tags/0.25.2 # later tags seem to break (haven't investigated further)
CC=cc CXX=CC python setup.py build
export PYTHONPATH=$INSTALL_AREA/lib64/python2.7/site-packages
CC=cc CXX=CC python setup.py install --prefix=$INSTALL_AREA

cd ../

# Custom build mpi4py with local compilers for Theta worker nodes
git clone git@github.com:mpi4py/mpi4py.git
cd mpi4py
git checkout tags/2.0.0 # later tags seem to break (haven't investigated further)
##  edit mpi.cfg
##  under [mpi]
##  uncomment mpicc = cc
##  uncomment mpicxx = cxx
CC=cc CXX=CC python setup.py build
CC=cc CXX=CC python setup.py install --prefix=/path/to/yoda

# Custom build yampl library
git clone git@github.com:vitillo/yampl.git
cd yampl
./configure --prefix=$INSTALL_AREA CC=cc CXX=CC
make -j 10 install
# in my case this fails in the zeromq folder
# so I did this
cd zeromq
./configure --prefix=$INSTALL_AREA CC=cc CXX=CC LDFLAGS=-dynamic
make -j 10 install
# this will also fail to build the shared library because it is trying
# to include the static library from zeromq so edit the Makefile to 
# LIBZMQ = ./zeromq/src/.libs/libzmq.so
# if that library does not exist, you may need to re-run the configure
# step in the zeromq folder with the option --enable-shared
# after you have zeromq build go back to yampl directory and build
cd ..
make -j 10 install
cd ..

# Custom build yampl python bindings
git clone git@github.com:vitillo/python-yampl.git
cd python-yampl
##  edit setup.py such that inside the 'include_dirs' inside the 'Extension' have
##  both the path to the '$INSTALL_AREA/yampl/install/include/yampl' 
##  and '$INSTALL_AREA/yampl/install/include'
##  in my case, I also had to add to the setup.py, at the top:
##  yampl_lib_path = subprocess.check_output(['pkg-config','--libs-only-L','yampl']).decode('utf-8')[:-2]
##  then the extension declaration looks like this:
##    Extension("yampl",
##              sources=["yampl.pyx"],
##              libraries=["yampl"],
##              library_dirs=[yampl_lib_path.replace('-L','')],
##              include_dirs=[yampl_include,yampl_include.replace('/yampl','')],
##              extra_link_args=[yampl_libs],
##              language="c++"),
python setup.py build_ext
python setup.py install
cd ..

# install yoda
pip install git+git://github.com:PanDAWMS/panda-yoda.git
# to later update yoda you must delete the $VIRTUAL_ENV/lib/python2.7/site-packages/pandayoda 
# and panda_yoda-0.0.1... folders by hand, then rerun the pip install command
```

# Installing on NERSC/Cori

``` bash
module load python/2.7-anaconda
module swap PrgEnv-intel PrgEnv-gnu

# follow directions for ALCF/Theta
```

# Configuration

After installation, you will find a `yoda_template.cfg` file in your `$VIRTUAL_ENV/etc` folder. This file uses the `ConfigParser` syntax from the module of the same name. 

Most subsections represents a different thread of Yoda. The generic `loop_timeout` parameter controls polling loops of running threads and can be finetuned for performance and responsiveness (60 seconds should generally be fast enough). 

## [Yoda]
- `wallclock_expiring_leadtime`: this is experimental at the moment, but in the future would allow the user to specify how many seconds before the expected wall-clock of the batch job will expire that Yoda should kill all MPI ranks and exit. Currently the wall-clock time is passed to Yoda on the command line. 

## [WorkManager]
- `send_n_eventranges`: controls the number of event ranges the `WorkManger` thread sends to Droids requesting more work. Current recommendation is to keep this value at the number of AthenaMP workers (128 on Theta, 136 on Cori, etc.). Multiplying by 2 or more could be recommended if long wall-clock times are anticipated.
- `request_n_eventranges`: controls the number of event ranges to request from Harvester when there are no more to hand out to Droid instances. Current recommendation is to keep this value at the number total AthenaMP workers in your batch job, i.e. Number of Nodes times number of AthenaMP workers per node. Again, multiplying by 2 or more could be recommeneded if longer wall-clock times are anticipated. 

## [RequestHarvesterJob]
- `messenger_plugin_module`: controls the plugin used to communicate with Harvester to request a job definition. Currently there is only one `pandayoda.yoda.shared_file_messenger`. In the future, there will likely also be sockets version.

## [RequestHarvesterEventRanges]
- `messenger_plugin_module`: controls the plugin used to communicate with Harvester to request a job definition. Currently there is only one `pandayoda.yoda.shared_file_messenger`. In the future, there will likely also be sockets version.

## [shared_file_messenger]
- `harvester_config_file`: this should point at the harvester 'cfg' file. the Shared File Messenger needs to know the names of the JSON files Harvester intends to use for communication.

## [Droid]
- `yampl_socket_name`: This is a string used to define the yampl socket name that is used to communicate with AthenaMP. It is placed on the command line of the transform (Sim_tf.py). Current setting is `EventServiceDroid_r{rank}` where `{rank}` takes advantage of the python string format command to replace rank with the ranknumber of the Droid running AthenaMP.
- `subprocess_stdout`: This controls the transforms output log file name. Current settings is - `transform_stdout_r{rank}_pid{PandaID}.log` where `{rank}` and `{PandaID}` take advantage of the python string format command to replace rank/PandaID with the ranknumber/PandaID of the Droid/Job running AthenaMP. Keep in mind that if you change this name you must change the Harvester plugin that grabs log data to upload to the grid.
- `subprocess_stderr`: same as above, but for the error stream. Currently set to `transform_stderr_r{rank}_pid{PandaID}.log`. 
- `working_path`: controls the name of the working path for each Droid instance. This is a relative path to the working directory inwhich Yoda is launched by Harvester. Current setting is `droid_rank_{rank}` where `{rank}` takes advantage of the python string format command to replace rank with the ranknumber of the Droid running AthenaMP.

## [FileManager]
- `output_file_type`: This is a setting for the JSON output for Harvester. This setting doesn't need to be changed and should always be `es_output`.
- `messenger_plugin_module`: controls the plugin used to communicate with Harvester to request a job definition. Currently there is only one `pandayoda.yoda.shared_file_messenger`. In the future, there will likely also be sockets version.
- `transfer_mode`: Currently ignored.

## [JobComm]
- `get_more_events_threshold`: threshold at which the JobComm will request more events from Yoda.

## [TransformManager]
- `template`: The full path to a template file for running the Transform. An example of this template is included for running Sim_tf on Theta in `templates/ThetaSubmitTF.sh`. Another version for running inside a singularity container is provided in `templates/ThetaSubmitSingularity.sh`. The template file contains python string format variables like `{rank}`, `{processingType}`, `{taskID}`, `{pandaID}`, `{release}`, `{package}`, `{cmtConfig}`, `{gcclocation}`, `{transformation}`, `{jobPars}`.
- `run_script`: The name of the script in which the `template` will be written.
- `use_mock_athenamp`: ignored for now. Ideally for testing yoda without running Athena, but yeah... who has time for that.
- `ATHENA_PROC_NUMBER`: Currently ignored. In the future, may override the number of AthenaMP workers on each node. 
- `use_container`: Set to `true` if you want the transform to run inside a container
- `container_prefix`: Used to set the container execute command. For Theta this looks like `/usr/bin/singularity exec -B /lus/theta-fs0/projects/AtlasADSP:/lus/theta-fs0/projects/AtlasADSP:rw /lus/theta-fs0/projects/AtlasADSP/centos6-cvmfs.atlas.cern.ch.img.x86_64-slc6-gcc49+.201803081026 /bin/bash`

## [MPIService]
- `default_message_buffer_size`: controls the MPI Message size limits in mpi4py. Current recommended value is `10000000`.




