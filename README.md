## Setup

### Option 1 : RPMs

Configure the YUM repository (only once), by running the following commands (on CC7, as root) :
```
rpm -q CERN-CA-certs || yum install CERN-CA-certs
cat > /etc/yum.repos.d/alisw-el7.repo <<EOF
[alisw-el7]
name=ALICE Software - EL7
baseurl=https://ali-ci.cern.ch/repo/RPMS/el7.x86_64/
enabled=1
gpgcheck=0
EOF
```
Configure Modules (only once), by adding to your `.bashrc`:

    export MODULEPATH=/opt/alisw/el7/modulefiles:$MODULEPATH

Search and install packages:

    yum search alisw-flp

You can then install it (change the version number appropriately):

    yum install -y alisw-flpproto+v0.1.1-1

Packages are not installed in system directories.
You’ll need to enable them explicitly, in your terminal or in your
`.bashrc` (adapt the version number):

    eval `modulecmd bash load flpproto/v0.1.1-1`

Errors saying something like:

    GCC-Toolchain/v4.9.3-alice3-2(16):ERROR:105: Unable to locate a modulefile for 'Toolchain/GCC-v4.9.3'

are irrelevant (the correct GCC is available, check with `which gcc`).

### Option 2 : alibuild

This is the recommended option for development.

Install devtoolset-6:

```
yum install centos-release-scl
yum install devtoolset-6
source /opt/rh/devtoolset-6/enable # to be added to your .bashrc
```

Setup alibuild (see [here](https://alisw.github.io/alibuild/o2-tutorial.html) for details):

```
sudo pip install alibuild==1.4.0 # or a later version
mkdir -p $HOME/alice
cd $HOME/alice
aliBuild init flpproto
```

Check what dependencies you are missing and you would like to add.

    aliDoctor --defaults o2-daq flpproto

Build dependencies and flp prototype

    aliBuild --defaults o2-daq build flpproto

Load environment to use or develop

    ALICE_WORK_DIR=$HOME/alice/sw; eval "`alienv shell-helper`" # Add it to .bashrc
    alienv load flpproto/latest

### Post installation items

```
sudo systemctl start mariadb
qcDatabaseSetup.sh
```

## QC Configuration

The QC requires a number of configuration items. An example config file is
provided in the repo under the name _example-default.ini_. Moreover, the
outgoing channel over which histograms are sent is defined in a JSON
file such as the one called _alfa.json_ in the repo.

**QC tasks** must be defined in the configuration :

```
[myTask_1]                      # define parameters for task myTask_1
taskDefinition=taskDefinition_1 # indirection to the actual definition
```

We use an indirect system to allow multiple tasks to share
most of their definition.

```
[taskDefinition_1]
className=AliceO2::QualityControlModules::Example::ExampleTask
moduleName=QcExample     # which library contains the class
cycleDurationSeconds=1
```

The JSON file contains a typical FairMQ device definition. One can
 change the or the address there:
```
{
    "fairMQOptions": {
        "devices": [
            {
                "id": "myTask_1",
                "channels": [
                    {
                        "name": "data-out",
                        "sockets": [
                            {
                                "type": "pub",
                                "method": "bind",
                                "address": "tcp://*:5556",
                                "sndBufSize": 10000,
                                "rcvBufSize": 10000,
                                "rateLogging": 0
                            }
                        ]
                    }
                ]
            }
        ]
    }
}
```

**QC checkers** are defined in the config file :

```
[checker_0]
broadcast=0 # whether to broadcast the plots in addition to storing them
broadcastAddress=tcp://*:5600
id=0        # only required item
```

And for the time, the mapping between checkers and tasks must be explicit :

```
[checkers] ; needed for the time being because we don't have an information service
numberCheckers=1
; can be less than the number of items in tasksAddresses
numberTasks=1
tasksAddresses=tcp://localhost:5556,tcp://localhost:5557,tcp://localhost:5558,tcp://localhost:5559
```

There are configuration items for many other aspects, for example the
database connection, the monitoring or the data sampling.

## Execution

### Basic
With the default config files provided, we run a qctask that processes
random data and forwards it to a GUI and a checker that will check and
store the MonitorObjects.

```
# Launch a task named myTask_1
qcTaskLauncher -n myTask_1 \
               -c file:///absolute/path/to/example-default.ini \
               --id myTask_1 \
               --mq-config path/to/alfa.json

# Launch a gui to check what is being sent
# don't forget to click the button Start !
qcSpy

# Launch a checker named checker_0
qcCheckerLauncher -c file:///absolute/path/to/example-default.ini -n checker_0
```

### Readout chain

1. Modify the config file of readout (Readout/configDummy.cfg) :
```
[readout]
exitTimeout=-1
(...)
[sampling]
# enable/disable data sampling (1/0)
enabled=1
# which class of datasampling to use (FairInjector, MockInjector)
class=FairInjector
```
2. Start Readout
```
readout.exe file:///absolute/path/to/configDummy.cfg
```
3. Modify the config file of the QC (QualityControl/example-default.ini) :
```
[DataSampling]
implementation=FairSampler
```
4. Launch the QC task
```
qcTaskLauncher -n myTask_1 \
               -c file:///absolute/path/to/configDummy.cfg \
               --id myTask_1 \
               --mq-config path/to/alfa.json
```
5. Launch the checker
```
qcCheckerLauncher -c file:///absolute/path/to/example-default.ini -n checker_0
```
6. Launch the gui
```
qcSpy file:///absolute/path/to/example-default.ini
# then click Start !
```
In the gui, one can change the source and point to the checker by
changing the path in the field at the bottom of the window (first click
`Stop` and `Start` after filling in the details). The port of the checker
is 5600.
One can also check what is stored in the database by clicking `Stop`
and then switching to `Database` source. This will only work if a config
file was passed to the `qcSpy` utility.

## Development notes

- ExampleTask should be outside the QualityControl package. It should be part of a QualityControlModule
  which would contain the skeleton for someone to develop their own task and checkers. qaLauncher would then
  instantiate the class in the library produced by the module.
- FindROOT.cmake should be replaced with the Config file of ROOT when compiled with CMake.
- Independent from O2 ideally. In the sense that it should be usable by other experiemnts.
- Need for TApplication ?
- The name of the task is the only required thing in the end as it should then link to all required info from the
  Config (module, class name, source, ...).
- Modules (user code) is in a different repo because the QC does not depend on AliceO2 but modules will.
  We must be able to compile a single module. For the time being it is a single project with subfolders for each
  module. If need be we can move to independent cmake projects.
- To test (from build dir):
            ./bin/qcTaskLauncher  -n myTask_1 -c file://$PWD/../QualityControl/example-default.ini
            ./bin/alfaTestReceiver --id receiver --mq-config ../QualityControl/alfaTestReceiver.json
            ./bin/qcCheckerLauncher -c file://$PWD/../QualityControl/example-default.ini -n checker_0
            ./bin/qcSpy
- To start DB :

## TODO

* add more checks to the example task to use the configuration in the isabovemean
* clean up code and documentation
* we give the taskName to most classes because it is important to get the config. Should we rather pass the config itself or make the Control a singleton that anyone can contact to get any of those two ?

## BENCHMARK

Master machine : has connection to all the other machines as root password less. (phaidgw02)
Update the list of machines in benchmark_machine_setup.sh. Run it.
update the list of task and checker machines in benchmark.sh. Also decide the test matrix.
Run benchmark.sh

## Misc

cmake -DCMAKE_MODULE_PATH="$SOURCEDIR/cmake/modules;$FAIRROOT_ROOT/share/fairbase/cmake/modules;$FAIRROOT_ROOT/share/fairbase/cmake/modules_old" -DFairRoot_DIR=$FAIRROOT_ROOT -DFAIRROOTPATH=$FAIRROOT_ROOT -DROOTSYS=$ROOTSYS -DBOOST_ROOT=$BOOST_ROOT ..
