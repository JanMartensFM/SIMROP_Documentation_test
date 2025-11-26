# SIMROP functionality

## Component Manager
![component_manager](https://github.com/user-attachments/assets/6cad59b4-2787-4e5b-aaa7-e8270709fdf4)


A **Component Manager** will handle the lifecycle of components of a certain type. We can for instance have a Robot Manager which handles the different robots available in the current setup or a [Task Manager](#Task%20Manager) which manages the task servers which will execute the [Behavior Tree tasks](#Behavior%20Tree%20Task).

A Component Manager can accept different [plugins](#Plugins) to change its configuration. Each Component Manager also has its own [Lifecycle Node](#Lifecycle%20Node) to cycle it through its different states.

### Configuring a Component Manager and its components
In order to make this flexible, multiple configuration files are present.

#### Configure which components will be instantiated

The first configuration file describes the components or plugins that will be instantiated. This is an example of such a file, in this case a task_plugin.yaml:

```yaml
plugins:
  - id: "task_1"
    type: "fm_sr_task::BehaviorTreeTask"
```

When a Task Manager is instantiated and its configure method is called, it will create a node with name **task_1** and of type **fm_sr_task::BehaviorTreeTask**.

#### Configure the components that will be instantiated

The components that are instantiated may require extra configuration settings, specific to their type. A second configuration file is present to allow this. This is for instance a task_configuration.yaml:

```yaml
task_1:
  libraries: 
    - fm_sr_robot_bt_lib
```

When the node with name **task_1** is configured, the matching node in the task_configuration.yaml file will be passed to the task. It is up to the task to read its configuration. In this way every component is free to retrieve the data that it needs. For a robot this may for instance be an IP address and a maximum speed.


#### Using a different configuration

A component manager has a default configuration. In an actual setup the configuration will probably not be the same as this default setup. So we need to be able to override the default configuration. When starting a component manager, a parameter config_package_name can be passed to specify which package contains it's configuration. This is the syntax to use:

```
ros2 run fm_sr_task task_manager --ros-args -p config_package_name:=fm_sr_robot
```

When the `task_manager` is configured, it will load its configuration files from the package `fm_sr_robot`.

The documentation of the Component Manager that you want to use should specify the names of the configuration files that it will use.

Note that a configuration package can contain configuration files for different Component Managers, since normally these files should be different. You could for instance have one central configuration package which is used by all the Component Managers that you will use, and then run them using:

```
ros2 run fm_sr_task task_manager --ros-args -p config_package_name:=palletizing_configuration
ros2 run fm_sr_robot robot_manager --ros-args -p config_package_name:=palletizing_configuration
```

## Context Manager
![geometric_context_model](https://github.com/user-attachments/assets/3c074d6a-67dd-4b62-bc6f-e584da54702f)

## Task Manager
![task_manager](https://github.com/user-attachments/assets/28462cec-f525-4db8-900d-6197d3abefee)

This is a specific type of [Component Manager](#Component%20Manager) for task specifications. It loads each of the provided task descriptions and schedules the different resources available to the system to perform each of these tasks. As shown in the diagram above, these resources can be shared between different tasks (e.g. a robot can switch between working on different tasks depending on when its functionality is needed). This resource scheduling is performed by the Task Manager without needing explicit input from the user, i.e. the user does not need to decide when each resource is being used on which task.

## Bringup and Lifecycle management
The main entities that comprise a good ROS2-based software stack are built on [**ROS2 Lifecycle Nodes**](https://design.ros2.org/articles/node_lifecycle.html). These nodes can be supervised by a [**Lifecycle Manager**](https://github.com/ros2/demos/blob/kilted/lifecycle/README.rst), which can actively transition the nodes under its supervision through their lifecycle states. These lifecycle states (as shown in the diagram below), are:

* **Unconfigured**: the node has been instantiated but none of its functionality has been initialized. This is the first state immediately after constructing the node or after a (fatal) error has occurred in its lifecycle management.
* **Inactive**: the node has been configured but is not currently performing any processing. In this state, the node can be modified without immediately impacting its behavior during operation. This state is reached after a successful transition from the *Unconfigured* state or from the *Active* state.
* **Active**: the node is running and exposes its functionality or runs its processes. The node remains in this state unless it is purposely deactived or shut down (through external input or errors) in which case it will return to its *Inactive* (you might want to restart the node) or *Finalized* (you have no further need of the node) states respectively.
* **Finalized**: the node has ended its lifecycle and has been terminated.

![lifecycle_node](https://github.com/user-attachments/assets/18c13465-4ad2-41f2-8cbc-8c6856933365)

### Manual lifecycle management
If lifecycle nodes are launched without automated lifecycle management, the user must manually transition the involved nodes through their lifecycle transitions to the operational state. An example with two lifecycle nodes:

```bash
$ ros2 run fm_sr_task task_manager
```

From another terminal, the user can have the control of the startup of the lifecycle node:
```bash
$ ros2 lifecycle set /task_manager configure
$ ros2 lifecycle set /task_manager activate
```

A similar procedure for conducting a controlled shutdown:
```bash
$ ros2 lifecycle set /task_manager deactivate
$ ros2 lifecycle set /task_manager cleanup
$ ros2 lifecycle set /task_manager shutdown
```

### Automated lifecycle management
In a larger context, the software stack will consist of multiple lifecycle nodes that need to be transitioned to their operational state. Therefore, the fm_sr_core package implements a lifecycle manager that will automatically transition the specified nodes to establish an automatic controlled startup of the software stack. To this end, you can compose a ROS2 launch file like the example below:

```bash
from launch_ros.actions import Node
from launch import LaunchDescription

def generate_launch_description():
    task_manager = Node(
        package='fm_sr_task',
        executable='task_manager',
        name='task_manager',
        output='screen'
    )
    
    context_manager = Node(
        package='fm_sr_context',
        executable='context_manager',
        name='context_manager',
        output='screen'
    )

    lifecycle_manager = Node(
        package='fm_sr_core',
        executable='lifecycle_manager',
        name='lifecycle_manager',
        output='screen',
        parameters=[{
            'managed_nodes': ['context_manager', 'task_manager']
        }]
    )

    return LaunchDescription([
        task_manager,
        context_manager,
        lifecycle_manager
    ])
```

This example launches two lifecycle nodes: `task_manager` and `context_manager`. However, when they are launched, they will remain in their unconfigured state. Along the lifecycle nodes, the launch file also runs the **lifecycle manager**, which will take care of the controlled startup. It will transition the nodes specified in the parameter "managed_nodes" to their operational state. Note that the lifecycle manager transitions the nodes in the order they appear in the list given, not necessarily in the order the nodes were launched.  

## Lifecycle Management and the Component Manager
The sequence diagrams below represent what happens when the lifecycle manager (either the automatic lifecycle manager, or a user manually adhering this role from a terminal) commands lifecycle transitions to a SIMROP component manager. For each of the transitions, the component manager performs some internal logic, and then iterates over the supervised plugins to call their internal transition callback:

![sd_lifecycle_management_startup](https://github.com/user-attachments/assets/f98a5824-4c81-44a5-948d-9c4c25186ef9)

![sd_lifecycle_management_shutdown](https://github.com/user-attachments/assets/676b83fe-dc7f-4d23-8f9d-be9171f2f747)


## Behavior Tree Task
![dmd-behavior_tree_task](https://github.com/user-attachments/assets/c0e2c53a-04a0-4a4c-a3b9-401f1a15e163)

This is a type of task which can be performed by employing the different resources available to the system. As the name suggests, the Behavior Tree task employs [behavior trees](https://en.wikipedia.org/wiki/Behavior_tree_(artificial_intelligence,_robotics_and_control)) to structure the task specification.

## Plugins
### Action Node Plugins
![service_client_action_node](https://github.com/user-attachments/assets/1b3a1c7c-d1a2-4db0-b29f-a923cf10a3d2)

### Task Plugins
![dmd-task_plugins](https://github.com/user-attachments/assets/0857bc06-3a3c-4b60-b0e9-b1eb3b9bb3dc)

## Simple Action Server
![simple_action_server](https://github.com/user-attachments/assets/5eadd39b-9955-4d23-b7f4-e0545a43efbe)

This is an implementation of a ROS2 action server, i.e. a node which offers some functionality and can be queried by an action client. The action server can then provide feedback to the client during execution of the query and upon its completion. 