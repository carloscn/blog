  
TUNA is a comprehensive software framework for analog signal data acquisition. As a software framework, it can be deployed on various platform devices to achieve data acquisition of analog signals. TUNA can be integrated into the soft core of self-developed scientific research experimental equipment, realizing end-to-end data transfer, recording, and extraction functions.

The basic architecture of TUNA is divided into three parts: the agent, server, and client. The agent is the terminal device for data, typically a low-power device with an ARM processor integrated with an operating system (Linux/RTOS), equipped with basic functionalities like offline data saving and active data collection. The server runs as a background service on the operating system's desktop, monitoring the agent's status and reading its data, freeing users from worrying about the transport protocol layer. The client side comes in various forms, including the interface program included in TUNA or commonly used scientific software like MATLAB or Python, supporting one-click data import into MATLAB/Python variables.

Besides its fundamental capabilities, TUNA also offers a certain degree of customization. TUNA will open some interfaces for users to implement, allowing easy integration of the TUNA framework into their own projects.

# 1. Software Arch

The software Arch figure is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202401021634817.png)

## 1.1 Host Side Arch

![](https://raw.githubusercontent.com/carloscn/images/main/typora202401031509291.png)


## 1.2 Device Side Arch




# 2. Use cases

There are some Tuna use cases for your reference. 

## 2.1 ADC Data Gatheration


