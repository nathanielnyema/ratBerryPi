# ratBerryPi
A Raspberry Pi based system for controlling liquid-reward delivery and basic cue presentation for animal behavioral experiments. The system is customizable and programmable with Python. Functionality is provided for interfacing with the hardware either locally on the Pi or remotely from a client device. We also provide an additional Python package, [pyBehavior](https://github.com/nathanielnyema/pyBehavior) which contains tools for building GUIs to interface with the system (remotely or locally) and for developing behavioral protocols which leverage the hardware.

## Overview
ratBerryPi was built for the general class of behavioral paradigms that involve an animal collecting a liquid reward from one of many end points. The system is built around a modified version of the open source [Poseidon Syringe Pump](https://pachterlab.github.io/poseidon/), which we use to supply fluid as reward to any of multiple reward end-points via a manifold of 3-way stopcocks with luer connectors and an array of media isolation solenoid valves. One valve on the manifold is connected to a reservoir which can be used to intermittently refill the syringe without unloading and reloading it. The reward end-points in this system are modules that are each fitted with a lickometer, speaker and LED. The reward modules plug into a central interface that sits as a HAT on the Raspberry Pi controlling the system. In theory, up to 32 modules can be connected to a given pi by stacking 4 interface HATs. All peripherals are accessible and programmable through a Python class we've defined called the RewardInterface. Users may access this interface either remotely or locally (see [Usage](#usage)). The codebase was developed with flexibility and configurability in mind such that skilled users may modify the hardware and software to meet their needs (see [Software Overview](docs/software_overview.md) and [Hardware Overview](docs/hardware_overview.md)).

[***PICTURE HERE***]

## Getting Started
In this repository we provide stl files for our modified version of the syringe pump and manufacturing files for printing our custom PCBs which control all peripherals. For build instructions use the following links:

- [Hardware Assembly](docs/hardware_assembly.md)
- [PCB Ordering](docs/pcb_ordering.md)
- [Electronics Assembly](docs/electronics_assembly.md)


### Software Installation - (Raspberry Pi)
Clone this repository then follow these steps to setup a raspberry pi for use with ratBerryPi:

1. If you haven't already, install miniforge3 by downloading using the appropriate installer which can be downloaded from [here](https://github.com/conda-forge/miniforge?tab=readme-ov-file#download).

2. Create a conda environment using the environment.yml file provided in this repo and activate it by running the following from the project directory:

```
conda env create -f environment.yml
conda activate ratBerryPi
```

3. Run the following to configure the pi for use with the adafruit-blinka library (modified from [here](https://learn.adafruit.com/circuitpython-on-raspberrypi-linux/installing-circuitpython-on-raspberry-pi)):

```
pip install --upgrade adafruit-python-shell
curl https://raw.githubusercontent.com/adafruit/Raspberry-Pi-Installer-Scripts/master/raspi-blinka.py| sudo -E env PATH=$PATH python -
```

4. After the reboot, re-navigate to the project directory, re-activate the conda environment and run the following to finish the installation:

```
pip install '.[pi]'
```

#### If using with pyBehavior
If interfacing with ratBerryPi through [pyBehavior](https://github.com/nathanielnyema/pyBehavior) directly on the pi you can install pyBehavior in this environment by first cloning the pyBehavior repo and then running the following from the cloned repository:

```
conda env update --name ratBerryPi --file environment.yml
pip install .
```

### Software Installation - (Client Device)
As detailed in the Usage section below, one mode of operation of this system is to configure the raspberry pi that is connected to the hardware to function as a server that clients may connect to in order to trigger rewards or cues. For this to work, the ratBerryPi python package must also be installed on the client device. To install ratBerryPi on a client device, simply clone this repository, navigate to the cloned repository and run `pip install .` while any desired virtual environment is activated.


## Usage
There are 2 main modes of operation for the ratBerryPi. The Raspberry Pi can be configured as a server that clients on other machines may connect to in order to run commands through the RewardInterface class. Alternatively, one may write a Python program on the raspberry pi itself which creates an instance of the RewardInterface class and invokes methods of the class to run a behavioral protocol.


### Local RewardInterface

Users may interact with the ratBerryPi through an instance of the RewardInterface class as follows:

```python
from ratBerryPi.interace import RewardInterface

rpi = RewardInterface()
rpi.start()

# do something

rpi.stop()
``` 

If nothing is passed to the constructor of the class, rpi will be an instance of the RewardInterface configured according to the config file located at ratBerryPi/config.yaml in the cloned repo (see [Configuration](docs/software_overview.md/#configuration); you shouldn't need to alter the config if using the hardware we provide). The constructor can also optionally take the following parameters which can be ignored by most:

- `on` (threading.Event) - threading event object that can be used to synchronize start and stopping the interface with any other threads which may be running in your program (for an example see `ratBerryPi/remote/server.py` where we use an event to stop the interface when we stop the server)
- `config_file` - path to an alternate config file
- `data_dir` - path to a directory to save any data produced by the interface after calling the `record` method. If not specified all data is saved at `~/.ratBerryPi/data`

Several instance methods are provided through `rpi` which allow the user interact with the hardware. For a full list of these methods see the help documentation for the RewardInterface class. This documentation can be accessed from a python terminal as follows:

```python
from ratBerryPi.interface import RewardInterface
help(RewardInterface)
```

### Server-Client Mode
We provide a cli for creating either a server or a cli via the command `ratBerryPi` which can be called from the command line. To start a server on the raspberry pi, simply run the command `ratBerryPi server`. This will expose a port on the raspberry pi for clients to connect to for running commands through the reward interface and broadcasting information about the state of the device.  By default it will bind port 5562 but this can be set as needed by passing the arguments `--port`. If using an interface other than the reward interface.

Users may connect to the server programmatically by first creating an instance of the `Client` class defined in `client.py`. For example:

```python
from ratBerryPi.client import Client

host = '123.456.789' # raspberry pi ip address
port = 5562

cl = Client(host, port)
```

Once you've created the client, the 2 most important methods of this class are `run_command` and `get`. 

- `run_command` provides an interface to run commands remotely through the interface running on the server. It takes as input 2 positional arguments, the first of which is a string indicating the name of a method in the interface class to run. The second argument is a dictionary specifying keyword arguments for this function. For information on available methods see the help documentation for the RewardInterface as discussed above


- `get` provides an interface to retrieve state information from the reward interface; it takes as input a string indicating the attribute of interface class you would like to get the value of. This string should be everything that would come after the period when directly accessing an instance of the interface class. for example, if we want the position of a pump named `pump1` when using the reward interface, the request would be `'pumps["pump1"].position'`.  


Both `run_command` and `get` further take as input an optional keyword argument `channel` which allows the user to specify the name of a 'channel' for communicating with the server. These channels are simply an abstraction for a connection to the server and allow users to isolate certain types of requests to avoid cross talk. For example, I may want to create an app using the reward interface where it would be useful to spawn a thread in the background to continuously monitor the position of the pump so I can print it for the user to see. It would be useful in this case to create a new channel specifically dedicated to these requests. Behind the scenes the server keeps these channels isolated by spawning separate threads for handling requests made on different channels. To create a new channel simply use the method of the client class `new_channel` which takes as input one positional argument which is the name of the new channel.

To start a client via the cli simply run the command `ratBerryPi client`. The default behavior of this method is to connect to a server hosted locally. If you would like to connect to a ratBerryPi server from a separate client device provide the `--host` argument as well as the `--port` argument if the server is being hosted on a non-default port. This should launch a cli where you can enter method names from the reward interface as commands followed by argument names and associated arguments separated by spaces. For example, if you wanted to run the trigger_reward method for a 1 mL reward on module1,  you would enter:

```
trigger_reward module module1 amount 1
```

There are 2 special commands: `get` and `exit`. `exit` will close the connection to the server and stop the cli. `get` is the cli equivalent to the get method described above.

## Operating the Pump and Manifold
The primary interface to the pump and valves is meant to be in software, however we provide some features to control the pump and fill valve manually to help with loading and unloading a syringe. Specifically, there are 3 buttons on the interface HAT that users should be familiar with: the flush button, the reverse button, and the fill valve button. The flush button allows you to manually advance the pump carriage sled forward. The reverse buton allows you to manually advance the pump carriage sled backwards. The fill valve button allows you to manually toggle the valve that goes to the reservoir (i.e. when the fill valve button is pressed this valve is open). In theory it is also possible to manually turn the lead screw to move the carriage sled if the pump is powered down (i.e. the 12V power supply to the interface board is unplugged) however we strongly discourage operating the pump in this way. In general it should be noted that whenever the 12V power supply is unplugged or the pump is unplugged, you should re-calibrate the pump before using it for anything. To do this you can use the calibrate method ot the Reward Interface.

### Filling the lines
For optimal performance, before triggering any rewards, all lines for reward delivery must be filled with the solution to be delivered to the reward ports. The key to doing this properly is making sure there are as few air bubbles in the lines as possible. We provide a method through the RewardInterface called fill_lines which automatically fills the lines. We strongly recommend users call this method before triggering any rewards. In the pyBehavior PumpConfig widget there is a button available that calls this method. Prior to running this method, be sure to pre-fill a syringe, load it in the pump, and make sure the reservoir is full. We've also found it useful to manually push some fluid to the reservoir while the pump is loaded with the full syringe before calling `fill_lines`. ***NOTE: If you do this make sure to hold open the fill valve with the fill valve button*** Importantly, this process can only work if the syringe used for filling the lines has at least as much volume as the dead volume leading up to the reservoir. Depending on the use case, it may even be in the user's best interest to use a fairly large syringe (about 30 mL) to fill the lines and switch to a more precise syringe for the experiment itself.