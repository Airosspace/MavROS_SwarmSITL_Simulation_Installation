
# MAVROS and SWARM SITL

MAVROS is a ROS (Robot Operating System) package that provides communication between ROS and the MAVLink protocol used by ArduPilot. SWARM SITL is a simulation environment that allows developers to test swarm behavior using multiple simulated drones or vehicles.

### MAVROS
MAVROS is a powerful tool that enables developers to create ROS nodes that communicate with the MAVLink protocol used by ArduPilot. It provides a bridge between ROS and the ArduPilot firmware, allowing developers to send commands and receive sensor data from ArduPilot using ROS.

MAVROS has several features that make it a powerful tool for developers. These features include:

- Support for multiple MAVLink dialects: MAVROS supports multiple MAVLink dialects, allowing it to communicate with a wide range of drones and vehicles.
- Support for multiple communication protocols: MAVROS supports multiple communication protocols, including TCP, UDP, and serial.
- Flexible architecture: MAVROS is designed to be flexible and extensible. It allows developers to create custom plugins and nodes that extend its functionality.
- Robust error handling: MAVROS includes robust error handling capabilities that help developers diagnose and fix communication issues.

### SWARM SIMULATION
SWARM SITL is a simulation environment that allows developers to test swarm behavior using multiple simulated drones or vehicles. It is built on top of the ArduPilot SITL and provides a framework for testing swarm behavior in a safe and controlled environment.

SWARM SITL has several features that make it a powerful tool for developers. These features include:

- Support for multiple vehicles: SWARM SITL can simulate multiple vehicles, allowing developers to test swarm behavior with different numbers and types of drones or vehicles.
- Real-time testing: SWARM SITL allows developers to test swarm behavior in real-time, without the need for physical drones or vehicles.
- Flexible configuration: SWARM SITL allows developers to configure a wide range of parameters for each simulated vehicle, including its starting position, velocity, and orientation.
- Support for custom behaviors: SWARM SITL provides a framework for creating custom swarm behaviors, allowing developers to test new algorithms and strategies.


## Installation

In this tutorial we are using **Ubuntu 20.04** and **ROS Noetic**

Code blocks are meant to be typed in Terminal windows. "Control+Alt+T" opens a new Terminal window.

### 1. Install ROS

   - Do _Desktop-full Install_
   - Follow until _Step 1.7_ at the end of the page

   First, install **ROS Noetic** using the following instructions: http://wiki.ros.org/noetic/Installation/Ubuntu


### 2. Set Up Catkin workspace

We use `catkin build` instead of `catkin_make`. Please install the following:
```
sudo apt-get install python3-wstool python3-rosinstall-generator python3-catkin-lint python3-pip python3-catkin-tools
pip3 install osrf-pycommon
```

Then, initialize the catkin workspace:
```
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws
catkin init
```

### 3. Dependencies installation

Install `mavros` and `mavlink` from source:
```
cd ~/catkin_ws
wstool init ~/catkin_ws/src

rosinstall_generator --upstream mavros | tee /tmp/mavros.rosinstall
rosinstall_generator mavlink | tee -a /tmp/mavros.rosinstall
wstool merge -t src /tmp/mavros.rosinstall
wstool update -t src
rosdep install --from-paths src --ignore-src --rosdistro `echo $ROS_DISTRO` -y

catkin build
```
Add a line to end of `~/.bashrc` by running the following command:
```
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
```

update global variables
```
source ~/.bashrc
```

install geographiclib dependancy 
```
sudo ~/catkin_ws/src/mavros/mavros/scripts/install_geographiclib_datasets.sh
```


### 4. Clone IQ Simulation ROS package 

```
cd ~/catkin_ws/src
git clone https://github.com/Intelligent-Quads/iq_sim.git
```
Our repository should now be copied to `~/catkin_ws/src/iq_sim/` (don't run this line. This is just saying that if you browse in the file manager, you will see those folders).

run the following to tell gazebo where to look for the iq models 
```
echo "GAZEBO_MODEL_PATH=${GAZEBO_MODEL_PATH}:$HOME/catkin_ws/src/iq_sim/models" >> ~/.bashrc
```

### 5. Build instructions
Inside `catkin_ws`, run `catkin build`:

```
cd ~/catkin_ws
catkin build
```
update global variables
```
source ~/.bashrc
```

## Usage for both Ardupilot and ROS Swarm

This tutorial shows you how to model and control a swarming using ardupilot and Gazebo

### Pre-rec

add the models folder in the iq_sim repo to the gazebo models path
```
echo "export GAZEBO_MODEL_PATH=${GAZEBO_MODEL_PATH}:$HOME/catkin_ws/src/iq_sim/models" >> ~/.bashrc
```


### Connecting Multiple Vehicles SITL to Gazebo

You should think about ardupilot as purely a control system. It takes sensor inputs and outputs commands to actuators. Gazebo is playing the role of a Flight Dynamics Model (FDM). The FDM encompasses all of the following, sensor models, actuator models and the dynamics of the vehicle. In real life ardupilot would communicate with your sensors via serial connections, but when you run SITL with gazebo ardupilot talks to the sensors and actuators via UDP connections (an IP protocol). Because of this we will need to specify different UDP ports in our robot models such that these streams do not conflict. 

Lets start by copying the drone1 folder in `iq_sim/models` and paste it as a copy. rename the folder to drone2.

then navigate to the drone2 folder open model.config and change the `<name>` tag to 
```
<name>drone2</name>
```

open the model.sdf and change the model name to `drone2` as well

scroll down to the ardupilot plugin and change
```
      <fdm_port_in>9002</fdm_port_in>
      <fdm_port_out>9003</fdm_port_out>
```
to 
```
      <fdm_port_in>9012</fdm_port_in>
      <fdm_port_out>9013</fdm_port_out>
```

Each successive drone fdm port should be incremented by 10 ie. 

drone3  
```
      <fdm_port_in>9022</fdm_port_in>
      <fdm_port_out>9023</fdm_port_out>
```
drone4
```
      <fdm_port_in>9032</fdm_port_in>
      <fdm_port_out>9033</fdm_port_out>
```
ect...


add the drone to `runway.world`
```
    <model name="drone2">
    <pose>10 0 0 0 0 0</pose>
     <include>
        <uri>model://drone2</uri>
      </include>
    </model>
```
launch the world 
```
roslaunch iq_sim runway.launch 
```

### launch ardupilot terminals 

to tell the ardupilot instance to incrament it's UPD connection use 
```
-I0 for drone1
-I1 for drone2
-I2 for drone3
ect ...
```

launch with 
```
sim_vehicle.py -v ArduCopter -f gazebo-iris --console -I0
```
```
sim_vehicle.py -v ArduCopter -f gazebo-iris --console -I1
```

### Connecting Multiple Drones to a Ground Station

Each Drone in your swarm will be producing mavlink messages. In order to discern what message is from which drone, we will need to assign each drone a unique system ID. This is controlled by the parameter `SYSID_THISMAV`. 

### Launching Ardupilot SITL Instances with Unique Parameters

Usually when we launch the ardupilot sitl simulation we launch the instance using the below command
```
sim_vehicle.py -v ArduCopter -f gazebo-iris --console
``` 
`-f` is used to specify the frame which Ardupilot will launch the simulation for. The frame type also specifies the location of the parameter files associated with the frame type. In order to simulate drones with different parameters we will need to create our own custom frames and parameter files.

First, we will want to edit the file `ardupilot/Tools/autotest/pysim/vehicleinfo.py` add the following lines in the SIM section.
```
            "gazebo-drone1": {
                "waf_target": "bin/arducopter",
                "default_params_filename": ["default_params/copter.parm",
                                            "default_params/gazebo-drone1.parm"],
            },
            "gazebo-drone2": {
                "waf_target": "bin/arducopter",
                "default_params_filename": ["default_params/copter.parm",
                                            "default_params/gazebo-drone2.parm"],
            },
            "gazebo-drone3": {
                "waf_target": "bin/arducopter",
                "default_params_filename": ["default_params/copter.parm",
                                            "default_params/gazebo-drone3.parm"],
            },
```
We will then need to create the following files

- `default_params/gazebo-drone1.parm`
- `default_params/gazebo-drone2.parm`
- `default_params/gazebo-drone3.parm`

each with their corresponding `SYSID_THISMAV` parameter value ie
- `default_params/gazebo-drone1.parm` should contain `SYSID_THISMAV 1`
- `default_params/gazebo-drone2.parm` should contain `SYSID_THISMAV 2`
- `default_params/gazebo-drone3.parm` should contain `SYSID_THISMAV 3`

#### Example gazebo-drone1.parm File
```
# Iris is X frame
FRAME_CLASS 1
FRAME_TYPE  1
# IRLOCK FEATURE
RC8_OPTION 39
PLND_ENABLED    1
PLND_TYPE       3
# SONAR FOR IRLOCK
SIM_SONAR_SCALE 10
RNGFND1_TYPE 1
RNGFND1_SCALING 10
RNGFND1_PIN 0
RNGFND1_MAX_CM 5000
SYSID_THISMAV 1
```

### Connecting Multiple Drones to qgroundcontrol

In order to connect multiple unique vehicles to a ground station, you will need to make sure that the TCP or UDP connection ports do not conflict. For this example we will be using a TCP connection. The first thing we need to do is relaunch our SITL ardupilot instances with a unique in/out TCP port for our GCS. 

```
sim_vehicle.py -v ArduCopter -f gazebo-drone1 --console -I0 --out=tcpin:0.0.0.0:8100 
```
```
sim_vehicle.py -v ArduCopter -f gazebo-drone2 --console -I1 --out=tcpin:0.0.0.0:8200 
```

- note 0.0.0.0 allows any device on out local network to connect to the ardupilot instance

## Contributing

We welcome contributions to AIROS Github! To contribute, please fork this repository, make your changes, and submit a pull request. Please ensure that your changes are well-documented and tested, and follow the `AIROS Github Project` coding guidelines.


## License

This Project is released under the `GNU General Public License v3.0.` Please see the LICENSE file for more information.

[AIROSSPACE](https://airosspace.com/)

![Logo](https://github.com/Airosspace/ROS_PX4_Installation/blob/main/images/logo.png)
## Credits

This Project is developed by the AIROS Software Development Team, a group of passionate software engineers dedicated to providing cutting-edge solutions for drone-based applications.
