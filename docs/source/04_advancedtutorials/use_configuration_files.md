# How to use configuration files from within your classes

## Introduction

When using a Manager component that derives from a [Component manager](#../03_tutorials/component_manager), you can use the provided configuration mechanisme.

As an example, we will take the Robot Manager which is available in the Resources package.

TODO => point to the usage of Configuration files, so don't repeat that here.


## File names

In order to receive information from configuration files, we first need to specify the names of the files that we want to use. 

Since we are inheriting from the ComponentManager class, we need to look at its constructor:

```cpp
ComponentManager(
    const std::string & node_name,
    const std::string & package_name,
    const std::string & plugins_yaml_file,
    const std::string & plugins_configuration_yaml_file,
    const rclcpp::NodeOptions & options)
```

For this tutorial, we are interested in the `plugins_yaml_file` and `plugins_configuration_yaml_file`.

The file `plugins_yaml_file` points to the file that will contain the different classes that will be instantiated by the ComponentManager that we are making, and looks like this:

```yaml
plugins:
  - id: "moveit_robot"
    type: "fm_sr_robot::MoveItRobot"

  - id: "input_output"
    type: "fm_sr_robot::FairinoInputOutput"
```

The ComponentManager class will take care of doing the necessary concerning instantiating these classes. Normally there is no need to override this behavior.

The file `plugins_configuration_yaml_file` will contain the actual parameters that will be used by the specific class that is being instantiated.

This is an example:

```yaml
moveit_robot:
  planning_group: "ur10e_arm_with_suction_gripper"
  planning_pipeline_id: "ompl"
  planner_id: "BFMTkConfigDefault"
  planning_time: 5.0
```

In this case, the class that will provision the action server listening to `moveit_robot`, of type `fm_sr_robot::MoveItRobot`, as specified in the `plugins_yaml_file`, will get the settings mentioned above. Note that the **id** in the `plugins_yaml_file` needs to match the root parameter in the `plugins_configuration_yaml_file`.

The parameters that are passed when overriding from ComponentManager specify the file that will be used. For instance for the RobotManager this looks like this:

```cpp
RobotManager(const rclcpp::NodeOptions & options)
  : ComponentManager(
    "robot_manager",
    "fm_sr_robot",
    "robot_plugins.yaml",
    "robot_configurations.yaml",
    options) {

    };
```

So that actual files that will be used are `robot_plugins.yaml` and `robot_configurations.yaml`. By using the mechanisme described in  "TODO JMA - point to using configuration", the location of these file can be determined.

## Reading the configuration

As described in a [Component Manager](../03_tutorials/component_manager.md), a class that derives from ComponentManager has a lifecycle. When it's `on_configure` method is called, it will configure components as specified in the `plugins_yaml_file` file. It will also read the   `plugins_configuration_yaml_file` file, and pass the related part of the yaml file to that specific component. In the example above, the class of type `fm_sr_robot::MoveItRobot`, will get the `moveit_robot` node from the configuration file.

The actual component classes inherit from `ComponentBase`  which has as configure method that looks like this:

```cpp
  virtual bool configure(
    const rclcpp_lifecycle::LifecycleNode::WeakPtr & parent_node,
    const std::string & component_name,
    const YAML::Node & configuration) = 0;
```

The last parameter is the yaml node, as made available by [yaml-cpp](https://github.com/jbeder/yaml-cpp). Follow the link for more information on how to use this library. In the example given above, the following code is used to read the `planning_group`:


```cpp
  if (YAML::Node parameter = configuration["planning_group"]) {

    move_group_interface_ = std::make_shared<MoveGroupInterface>(client_node_,
        parameter.as<std::string>());

  } else {
    logger_.error("Could not configure node {}, parameter planning_group is missing.",
        component_name);

    return false;
  }
```

