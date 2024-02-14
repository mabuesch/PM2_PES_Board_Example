# Line Follower Array 
The sensor incorporates eight diodes for line detection, with each diode's illumination indicating the presence of a line beneath it. Additionally, a calibration knob enables users to adjust detection sensitivity, accommodating variations in substrate and line color shades.

<center><img src="../images/line_follower_sensor.png" alt="Line follower sensor" width="600" /></center>

## Technical Specifications

| | Sparkfun Line follower sensor array |
|-|-|
|Sensor eyes number    | 8           |
|Interface             | I2C         |
|Supply Voltage        | 5 V         |
|Supply Current        | 25 - 185 mA |
|Read Cycle TIme       | 3.2 ms      |

## Links

[Sparkfun Line follower sensor array][1] <br>
[Sparkfun Line follower hookup guide][2]

## Datasheets

[Sparkfun Line follower sensor array](../datasheets/line_follower_array.pdf)

## Practical Tips
- <b>Before attempting to connect the sensor, it is crucial to carefully review the tab labeled "Connection to the PES-board" This sensor is highly sensitive, and mishandling during the connection process can lead to burnout. Therefore, thorough understanding and caution are necessary to avoid damaging the sensor. </b>

## Line follower driver
The ``line follower`` driver is used to calculate wheel speed based on sensor indications and robot geometry, as well as on other settings declared by the user.

To start using the ``line follower`` driver, the initial step in the ``main`` file is to create the line follower object and specify the pins to which the object will be assigned.

To set up the task properly, within the main function, it's assumed that you'll define two DC motor objects. To achieve this, please adhere to the instructions provided in document [DC motor](../markdown/dc_motor.md). Code snipets that should be placed in correct places:
```
#include "pm2_drivers/DCMotor.h"
#include "pm2_drivers/EncoderCounter.h"
```
```
DigitalOut enable_motors(PB_ENABLE_DCMOTORS); // create DigitalOut object to enable dc motors
```
```
// In the robot there will be used 78:1 Metal Gearmotor 20Dx44L mm 12V CB
// Define variables and create DC motor objects
const float voltage_max = 12.0f;
const float gear_ratio = 78.125f; 
const float kn = 180.0f / 12.0f; //motor constant rpm / V
DCMotor motor_M1(PB_PWM_M1, PB_ENC_A_M1, PB_ENC_B_M1, gear_ratio, kn, voltage_max);
DCMotor motor_M2(PB_PWM_M2, PB_ENC_A_M2, PB_ENC_B_M2, gear_ratio, kn, voltage_max);
```

**NOTE:**
- follow the instructions for the M2 motor ( used Closed-Loop Velocity Control (rotations per second))
- for this algorithm motion planner needs to be disabled

### Connection to the PES-board
---------------------------
For sensor communication, I2C technology is utilized, which relies on data pin and a clock pin (more information [here][3]). To power the sensor, a voltage of around 5V is required.

In the described robot, the following pins are utilized:
- Data **PB_9**
- Clock **PB_8**

[Nucleo Board pinmap][4]

## **WARNING!**
<b>As previously emphasized, this sensor is highly sensitive, and improper connections can lead to damage. It's crucial to thoroughly examine the provided pictures illustrating the correct method of connecting the sensor to avoid any issues.</b> <br>

To plug the power source you will need to use:
- two m/f jumper wires (black and red)
<center><img src="../images/mf_line_follower_array_connection.png" alt="Line follower sensor mf connection" width="600" /></center>

- two f/f jumpre wires (black and red)
<center><img src="../images/ff_line_follower_array_connection.png" alt="Line follower sensor ff connection" width="600" /></center>

<b> Take note of the pin descriptions on the sensor. Connect the red power cable to the pin labeled 5V and the black ground cable to the pin labeled GND.</b>
<center><img src="../images/line_follower_sensor_look.png" alt="Line follower sensor look from above" width="600" /></center>


### Create line follower object
---------------------------

Initially, it's essential to add the suitable driver to our main file and then create an object with the following variables defined (in **SI** units if applicable):
- SDA pin - used for data transfer in I2C standard
- SCL pin - used for clock synchronization
- bar_dist - distance of diodes from wheel axes
- d_wheel - wheel diameter
- L_wheel - wheelbase
- max_motor_vel_rps - maximum engine speed given in revolutions per second

The remaining values are defined by default, but there is a possibility to change some of the parameters, as described below the description of the internal algorithm operation.

```
#include "pm2_drivers/LineFollower.h"
```
```
const float d_wheel = 0.0564f;        // wheel diameter
const float L_wheel = 0.13f;          // distance from wheel to wheel
// sensor data evalution
const float bar_dist = 0.083f;

LineFollower lineFollower(PB_9, PB_8, bar_dist, d_wheel, L_wheel, motor_M2.getMaxPhysicalVelocity());
```

**NOTE:** 
- The velocity values provided as input are originally expressed in revolutions per second. However, within the driver, these values are converted into radians per second for calculation purposes. Once the calculation is completed in this unit, it is then converted back into revolutions per second. This conversion allows for the use of a unit directly compatible with the DC motor object.

### Parametres adjustment
---------------------------
The LineFollower class provides functionality to dynamically adjust key parameters:

1. Proportional Gain (Kp) and Non-linear Gain (Kp_nl):
- Function: `void setRotationalVelocityGain(float Kp, float Kp_nl)`
- Parameters: `Kp` and `Kp_nl`
- Description: These parameters influence the proportional and non-linear correction in the robot's angular velocity control, allowing users to fine-tune the response to deviations from the desired line angle.

2. Maximum Wheel Velocity:
- Function: `void setMaxWheelVelocity(float max_wheel_vel)`
- Parameter: `max_wheel_vel`
- Description: This parameter limits the maximum wheel velocity (argument in rotations per second), indirectly affecting the robot's linear and angular velocities. Users can adjust this limit to optimize performance based on environmental conditions and specific requirements.

### Driver ussage
---------------------------
The mathematical operations carried out within the driver determine the speed values for each wheel: right and left. These speed values are expressed in revolutions per second (RPS), allowing direct control of the motors using these values. Below is the code that should be executed when the **USER** button is pressed.
```
enable_motors = 1;
motor_M1.setVelocity(lineFollower.getRightWheelVelocity()); // set a desired speed for speed controlled dc motors M1
motor_M2.setVelocity(lineFollower.getLeftWheelVelocity());  // set a desired speed for speed controlled dc motors M2
```
**NOTE:** 
- Make sure you assign the speed to the correct motor (right/left)

Below, you'll find an in-depth manual detailing the inner workings of the driver functions. While it's not mandatory to use this manual, familiarizing yourself with its contents will enable more informed programming. For enhanced comprehension, it's recommended to refer to the [kinematics](../markdown/kinematics.md) document, which provides explanations of the mathematical operations involved.

### Thread algorithm description
---------------------------
The `followLine` function is a thread task method responsible for controlling the robot to follow a line based on sensor readings.

1. Thread Execution: The `followLine` method runs continuously in a thread loop. It waits for a thread flag to be set before executing, indicating that it should perform its task.

2. Sensor Reading: Inside the loop, the method checks if any LEDs on the sensor bar are active. If any LEDs are active, it calculates the average angle of the detected line segments relative to the robot's orientation. This angle is stored in `m_sensor_bar_avgAngleRad`.

3. Control Calculation:
- The maximum wheel velocity in radians per second (`m_max_wheel_vel_rads`) is calculated based on the maximum wheel velocity (`m_max_wheel_vel`).
- The rotational velocity (`m_robot_coord(1)`) is determined using a control function `ang_cntrl_fcn`, which adjusts the robot's orientation to align with the detected line.
- The translational velocity (`m_robot_coord(0)`) is determined using another control function `vel_cntrl_v2_fcn`, which calculates the robot's forward velocity based on the rotational velocity and geometric parameters.
- The robot's wheel speeds are calculated using the inverse transformation matrix `m_Cwheel2robot`.

4. Wheel Velocity Conversion: The calculated wheel speeds are converted from radians per second to revolutions per second (`m_right_wheel_velocity` and `m_left_wheel_velocity`).

5. Thread Flag Signaling: After completing its task, the method waits for the next iteration by waiting for the thread flag to be set again.

#### Angular velocity controller
-------------------------------
The `ang_cntrl_fcn` function is responsible for calculating the angular velocity of the robot based on the detected angle of the line relative to the robot's orientation. This function uses proportional and non-linear control to calculate the velocity based on mentioned angle.

1. Input Parameters:
- `Kp`: Proportional gain parameter for angular control.
- `Kp_nl`: Non-linear gain parameter for angular control.
- `angle`: The angle of the detected line relative to the robot's orientation.

2. Calculation:
- If the angle is positive (`angle > 0`), the function calculates the angular velocity (`retval`) using the formula:
    ```
    retval = Kp * angle + Kp_nl * angle * angle
    ```
    This formula applies proportional control (`Kp * angle`) along with a non-linear correction term (`Kp_nl * angle * angle`).
- If the angle is zero or negative (`angle <= 0`), the function calculates the angular velocity (`retval`) using a similar formula but with a negative sign for the non-linear term:
    ```
    retval = Kp * angle - Kp_nl * angle * angle
    ```
3. Output:
- The function returns the calculated angular velocity (`retval`).

#### Linear velocity controller
-------------------------------
The `vel_cntrl_v2_fcn` function calculates the linear velocity of the robot based on its angular velocity and geometric parameters. The function ensures that one of the robot's wheels always turns at maximum velocity while the other adjusts its speed to maintain the desired angular velocity.

1. Input Parameters:
- `wheel_speed_max`: Maximum wheel speed in radians per second.
- `b`: Geometric parameter related to the distance between the wheels .
- `robot_omega`: Angular velocity of the robot.
- `Cwheel2robot`: Transformation matrix from wheel to robot coordinates.

2. Calculation:
- If `robot_omega` is positive, it assigns the maximum wheel speed to the first wheel and calculates the speed for the second wheel by subtracting `2 * b * robot_omega` from the maximum speed.
- If `robot_omega` is negative or zero, it assigns the maximum wheel speed to the second wheel and calculates the speed for the first wheel by adding `2 * b * robot_omega` to the maximum speed.
- The function then calculates the robot's coordinate velocities by multiplying the transformation matrix `Cwheel2robot` by the wheel speeds.
- Finally, it returns the linear velocity of the robot, which corresponds to the velocity along the x-axis in the robot's coordinate system.

3. Output:
- The function returns the calculated linear velocity of the robot.

## Solution

[Line follower](../solutions/line_follower.txt)


<!-- Links: -->
[1]: https://www.sparkfun.com/products/13582
[2]: https://learn.sparkfun.com/tutorials/sparkfun-line-follower-array-hookup-guide
[3]: https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/all
[4]: https://os.mbed.com/platforms/ST-Nucleo-F446RE/