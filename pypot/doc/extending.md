# Extending pypot

While pypot has been originally designed for controlling dynamixel based
robots, it became rapidly obvious that it would be really useful to
easily:

-   control other types of motor (e.g. servo-motors controlled using
    PWM)
-   control an entire robot composed of different types of motors (using
    dynamixel for the legs and smaller servo for the hands for instance)

While it was already possible to do such things in pypot, the library
has been partially re-architectured in version 2.x to better reflect
those possibilities and most importantly make it easier for contributors
to add the layer needed for adding support for other types of motors.

> **note**
>
> While in most of this documentation, we will show how support for
> other motors can be added, similar methods can be applied to also
> support other sensors.

The rest of this section will describe the main concept behing pypot's
architecture and then give examples of how to extend it.

## Writing a new IO

In pypot's architecture, the IO aims at providing convenient methods to
access (read/write) value from a device - which could be a motor, a
camera, or a simulator. It is the role of the IO to handle the
communication:

-   open/close the communication channel,
-   encapsulate the protocol.

For example, the [pypot.dynamixel.io.io.DxlIO](pypot.dynamixel.io.html#pypot.dynamixel.io.io.DxlIO) (for dynamixel buses)
open/closes the serial port and provides high-level methods for sending
dynamixel packet - e.g. for getting a motor position. Similarly, writing
the [pypot.vrep.io.VrepIO](pypot.vrep.html#pypot.vrep.io.VrepIO) consists in opening the communication socket
to the V-REP simulator (thanks to [V-REP's remote API](http://www.coppeliarobotics.com/helpFiles/en/remoteApiFunctionsPython.htm))
and then encapsulating all methods for getting/setting all the simulated
motors registers.

> **warning**
>
> While this is not by any mean mandatory, it is often a good practice
> to write all IO access as synchronous calls. The higher-level
> synchronization loop is usually written as a
> [pypot.robot.controller.AbstractController](pypot.robot.html#pypot.robot.controller.AbstractController).

The IO should also handle the low-level communication errors. For
instance, the [pypot.dynamixel.io.io.DxlIO](pypot.dynamixel.io.html#pypot.dynamixel.io.io.DxlIO) automatically handles the
timeout error to prevent the whole communication to stop.

> **note**
>
> Once the new IO is written most of the integration into pypot should
> be done! To facilitate the integration of the new IO with the higher
> layer, we strongly recommend to take inspiration from the existing IO
> - especially the [pypot.dynamixel.io.io.DxlIO](pypot.dynamixel.io.html#pypot.dynamixel.io.io.DxlIO) and the
> [pypot.vrep.io.VrepIO](pypot.vrep.html#pypot.vrep.io.VrepIO) ones.

## Writing a new Controller

A [pypot.robot.controller](pypot.robot.html#pypot.robot.controller.AbstractController) is basically a synchronization
loop which role is to keep up to date the state of the device and its
"software" equivalent - through the associated IO.

In the case of the [pypot.dynamixel.controller.DxlController](pypot.dynamixel.html#pypot.dynamixel.controller.DxlController), it runs a
50Hz loop which reads the actual position/speed/load of the real motor
and sets it to the associated register in the
[pypot.dynamixel.motor.DxlMotor](pypot.dynamixel.html#pypot.dynamixel.motor.DxlMotor). It also reads the goal
position/speed/load set in the [pypot.dynamixel.motor.DxlMotor](pypot.dynamixel.html#pypot.dynamixel.motor.DxlMotor) and
sends them to the "real" motor.

As most controller will have the same general structure - i.e. calling a
sync. method at a predefined frequency - pypot provides an abstract
class, the [pypot.robot.controller.AbstractController](pypot.robot.html#pypot.robot.controller.AbstractController), which does
exactly that. If your controller fits within this conception, you should
only have to overide the
[pypot.robot.controller.AbstractController.update](pypot.robot.html#pypot.robot.controller.AbstractController) method.

In the case of the [pypot.vrep.controller.VrepController](pypot.vrep.html#pypot.vrep.controller.VrepController), the update
loop simply retrieves each motor's present position and send the new
target position. A similar approach is used to retrieve values form
V-REP sensors.

> **note**
>
> Each controller can run at its own pre-defined frequency and live
> within its own thread. Thus, the update never blocks the main thread
> and you can used tight synchronization loop where they are needed
> (e.g. for motor's command) and slower one when latency is not a big
> issue (e.g. a temperature sensor).

## Integrate it into the Robot

Once you have defined your Controller, you most likely want to define a
convenient factory functions (such as [pypot.robot.config.from\_config](pypot.robot.html#pypot.robot.config.from_config)
or [pypot.vrep.from\_vrep](pypot.vrep.html#pypot.vrep.from_vrep)) allowing users to easily instantiate their
[pypot.robot.robot.Robot](pypot.robot.html#pypot.robot.robot.Robot) with the new Controller.

By doing so you will permit them to seamlessly uses your interface with
this new device without changing the high-level API. For instance, as
both the [pypot.dynamixel.controller.DxlController](pypot.dynamixel.html#pypot.dynamixel.controller.DxlController) and the
[pypot.vrep.controller.VrepController](pypot.vrep.html#pypot.vrep.controller.VrepController) only interact with the
[pypot.robot.robot.Robot](pypot.robot.html#pypot.robot.robot.Robot)  through getting and setting values into
[pypot.robot.motor.Motor](pypot.robot.html#pypot.robot.motor.Motor)  instances, they can be directly switch.