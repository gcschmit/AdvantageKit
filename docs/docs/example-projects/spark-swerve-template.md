# Spark Swerve Template

AdvantageKit includes two swerve project templates with built-in support for advanced features:

- High-frequency odometry
- On-controller feedback loops, including MAXMotion for turn control
- Physics simulation
- Automated characterization routines
- Dashboard alerts for disconnected devices
- Pose estimator integration (not including vision)
- **Deterministic replay** with a **guarantee of accuracy**

By default, the Spark version of the swerve template is configured for robots with **MAXSwerve modules, four NEO Vortex drive motors, four NEO 550 turn motors, four duty cycle absolute encoders, and a NavX or Pigeon 2 gyro**. See the [TalonFX Swerve Template](talonfx-swerve-template.md) for swerve robots using Talon FX.

:::info
The AdvantageKit swerve templates are **open-source** and **fully customizable**:

- **No black boxes:** Users can view and adjust all layers of the swerve control stack.
- **Customizable:** IO implementations can be adjusted to support any hardware configuration (see the [customization](#customization) section).
- **Replayable:** Every aspect of the swerve control logic, pose estimator, etc. can be replayed and logged in simulation using AdvantageKit's deterministic replay features with _guaranteed accuracy_.

:::

## Setup

:::warning
This example project is part of the 2025 AdvantageKit beta release. If you encounter any issues during setup, please [open an issue](https://github.com/Mechanical-Advantage/AdvantageKit/issues).
:::

1. Download the Spark swerve template project from the AdvantageKit release on GitHub and open it in VSCode.

2. Click the WPILib icon in the VSCode toolbar and find the task `WPILib: Set Team Number`. Enter your team number and press enter.

3. Navigate to `src/main/java/frc/robot/subsystems/drive/DriveConstants.java` in the AdvantageKit project.

4. Update the values of `driveMotorReduction` and `turnMotorReduction` based on the robot's module type and configuration. This information can typically be found on the product page for the swerve module. These values represent reductions and should generally be greater than one.

5. Update the values of `trackWidth` and `wheelBase` based on the distance between the left-right and front-back modules (respectively)

6. Update the value of `wheelRadiusMeters` to the theoretical radius on each wheel. This value can be further refined as described in the "Tuning" section below.

7. Update the value of `maxSpeedMetersPerSec` to the theoretical max speed of the robot. This value can be further refined as described in the "Tuning" section below.

8. Set the value of `pigeonCanId` to the correct CAN ID of the Pigeon 2 (as configured using Tuner X). **If using a NavX instead of a Pigeon 2, see the [customization](#customization) section below.**

9. For each module, set the values of `...DriveMotorId` and `...TurnMotorId` to the correct CAN IDs of the drive Spark FLex and turn Spark Max (as configured in the REV Hardware Client).

10. For each module, set the value of `...ZeroRotation` to `new Rotation2d(0.0)`.

11. Deploy the project to the robot and connect using AdvantageScope. Verify the following:

    - There are no dashboard alerts or errors in the Driver Station console.

    - Manually rotate each module such that the position in AdvantageScope (`/Drive/Module.../TurnPosition`) is **increasing**. The module should be rotating **counter-clockwise** as viewed from above the robot. Verify that the units visible in AdvantageScope (radians) match the physical motion of the module.

    - Manually rotate each drive wheel and view the position in AdvantageScope (`/Drive/Module.../DrivePositionRad`) is **increasing**. Verify that the units visible in AdvantageScope (radians) match the physical motion of the module.

12. Manually rotate each module to align it directly forwards. **Verify using AdvantageScope that the drive position increases when the wheel rotates such that the robot would be propelled forwards.**

13. Record the value of `/Drive/Module.../TurnAbsolutePosition` for each aligned module. Update the value of `...ZeroRotation` for each module to `new Rotation2d(<insert value>)`.

## Tuning

### Feedforward Characterization

The project includes default [feedforward gains](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/introduction-to-feedforward.html#introduction-to-dc-motor-feedforward) for velocity control of the drive motors (`kS` and `kV`).

The project includes a simple feedforward routine that can be used to quicly measure the drive `kS` and `kV` values without requiring [SysId](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/system-identification/index.html):

1. Place the robot in an open space.

2. Select the "Drive Simple FF Characterization" auto routine.

3. Enable the robot in autonomous. The robot will slowly accelerate forwards, similar to a SysId quasistic test.

4. Disable the robot after at least ~5-10 seconds.

5. Check the console output for the measured `kS` and `kV` values, and copy them to the `driveKs` and `driveKv` constants in `DriveConstants.java`.

:::info
The feedforward model used in simulation can be characterized using the same method. **Simulation gains are stored in the `driveSimKs` and `driveSimKv` constants.**
:::

Users who wish to characterize acceleration gains (`kA`) can choose to use the full [SysId](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/system-identification/index.html) application. The project includes auto routines for each of the four required SysId tests. Two options are available to load data in SysId:

- The project is configured to use [URCL](https://docs.advantagescope.org/more-features/urcl) by default. This data can be exported as described [here](https://docs.advantagescope.org/more-features/urcl#sysid-usage).
- Export the AdvantageKit log file as described [here](../sysid-compatibility.md).

### Wheel Radius Characterization

The effective wheel radius of a robot tends to change over time as wheels are worn down, swapped, or compress into the carpet. This can have significant impacts on odometry accuracy. We recommend regularly recharacterizing wheel radius to combat these issues.

The project includes an automated wheel radius characterization routine, which only requires enough space for the robot to rotate in place.

1. Place the robot on carpet. Characterizing on a hard floor may produce errors in the measurement, as the robot's effective wheel radius is affected by carpet compression.

2. Select the "Drive Wheel Radius Characterization" auto routine.

3. Enable the robot is autonomous. The robot will slowly rotate in place.

4. Disable the robot after at least one full rotation.

5. Check the console output for the measured wheel radius, and copy the value to `wheelRadiusMeters` in `DriveConstants.java`.

### Drive/Turn PID Tuning

The project includes default gains for the drive velocity PID controllers and turn position PID controllers, which can be found in the "Drive PID configuration" and "Turn PID configuration" sections of `DriveConstants.java`. These gains should be tuned for each robot.

:::tip
More information about PID tuning can be found in the [WPILib documentation](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/introduction/introduction-to-pid.html#introduction-to-pid).
:::

We recommend using AdvantageScope to plot the measured and setpoint values while tuning. Measured values are published to the `/RealOutputs/SwerveStates/Measured` field and setpoint values are published to the `/RealOutputs/SwerveStates/SetpointsOptimized` field.

:::info
The PID gains used in simulation can be tuned using the same method. **Simulation gains are stored separately from "real" gains in `DriveConstants.java`.**
:::

### Max Speed Measurement

The effective maximum speed of a robot is typically slightly less than the theroetically max speed based on motor free speed and gearing. To ensure that the robot remains controllable at high speeds, we recommend measuring the effective maximum speed of the robot.

1. Set `maxSpeedMetersPerSec` in `DriveConstants.java` to the theoretical max speed of the robot based on motor free speed and gearing. This value can typically be found on the product page for your chosen swerve modules.

2. Place the robot in a open space.

3. Plot the measured robot speed in AdvantageScope using the `/RealOutputs/SwerveChassisSpeeds/Measured` field.

4. In teleop, drive forwards at full speed until the robot velocity is no longer increasing.

5. Record the maximum velocity achieved and update the value of `maxSpeedMetersPerSec`.

### Slip Current Measurement

The value of `driveMotorCurrentLimit` can be tuned to avoid slipping the wheels.

1. Place the robot against the solid wall.

2. Using AdvantageScope, plot the current of a drive motor from the `/Drive/Module.../DriveCurrentAmps` key, and the velocity of the motor from the `/Drive/Module.../DriveVelocityRadPerSec` key.

3. Accelerate forwards until the drive velocity increases (the wheel slips). Note the current at this time.

4. Update the value of `driveMotorCurrentLimit` to this value.

### PathPlanner Configuration

The project includes a built-in configuration for [PathPlanner](https://pathplanner.dev), located in the constructor of `Drive.java`. You may wish to manually adjust the following values:

- Robot configuration in the PathPlanner GUI ([docs](https://pathplanner.dev/robot-config.html)).
- Drive PID constants as configured in `AutoBuilder`.
- Turn PID constants as configured in `AutoBuilder`.

## Customization

### Setting Odometry Frequency

By default, the project runs at **100Hz**. This value is stored as `odometryFrequency` at the top of `DriveConstants.java` and can be changed. The project configures all devices to minimize CAN bus utilization, but we recommend monitoring utilization carefully when increasing frequency.

### Switching Between Spark Max and Flex

Switching between the Spark Max and Spark Max for drive and turn motors is very simple. In the constructor of `ModuleIOSpark`, change the call instantiating the Spark object to use `CANSparkMax` or `CANSparkFlex`. The configuration object must also be changed to the corresponding `SparkMaxConfig` or `SparkFlexConfig` class.

### Custom Gyro Implementations

The project defaults to the Pigeon 2 gyro, but can be integrated with any standard gyro. An example implementation for a NavX is included.

To change the gyro implementation, switch `new GyroIOPigeon2()` in the `RobotContainer` constructor to any other implementation. For example, the `GyroIONavX` implementation is pre-configured to use a NavX connected to the MXP SPI port. See the page on [IO interfaces](../recording-inputs/io-interfaces.md) for more details on how hardware abstraction works.

The `SparkOdometryThread` class reads high-frequency gyro data for odometry alongside samples from drive encoders. This class supports both Spark devices and generic signals. Note that the gyro should be configured to publish signals at the same frequency as odometry. Call `registerSignal` with a double supplier to create a queue, as shown in the `GyroIONavX` implementation:

```java
Queue<Double> yawPositionQueue = SparkOdometryThread.getInstance().registerSignal(navX::getAngle);
```

:::info
Reference the full `GyroIONavX` implementation for an example of how to create a timestamp queue and update the odometry inputs for the gyro.
:::

### Custom Module Implementations

The implementation of `ModuleIOSpark` can be freely customized to support alternative hardware configurations, such as using a TalonFX-based drive motor. When integrating with TalonFX devices, we recommend referencing the implementation found in the `ModuleIOTalonFX` class of the [TalonFX Swerve Template](talonfx-swerve-template.md).

As described in the previous section, the `SparkOdometryThread` supports non-Spark signals through the `registerSignal` method. This allows devices from different vendors to be freely mixed.

By default, the project uses a duty cycle encoder connected to a turn Spark Max. When using another absolute encoder (such as a CANcoder or HELIUM Canandmag), we recommend reseting the relative encoder based on the absolute encoder; the relative encoder can then be used for PID control. In this case, the following changes are required:

1. Create the encoder object in `ModuleIOSpark` and configure it appropriately.

2. Change the feedback sensor source of the turn controller:

```java
turnEncoder = turnSpark.getEncoder(); // Change the type of turnEncoder to RelativeEncoder
turnConfig.closedLoopConfig.feedbackSensor(FeedbackSensor.kMainEncoder); // Was: kAbsoluteEncoder
```

3. Incorporate the turn motor reduction in the encoder position and velocity factors:

```java
public static final double turnEncoderPositionFactor = 2 * Math.PI / turnMotorReduction; // Rotor Rotations -> Wheel Radians
public static final double turnEncoderVelocityFactor = (2 * Math.PI) / 60.0 / turnMotorReduction; // Roto RPM -> Wheel Rad/Sec
```

4. Reset the relative encoder position at startup:

```java
tryUntilOk(turnSpark, 5, () -> turnEncoder.setPosition(customEncoder.getPositionRadians()));
```

### Vision Integration

The `Drive` subsystem uses WPILib's [`SwerveDrivePoseEstimator`](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/estimator/SwerveDrivePoseEstimator.html) class for odometry updates. The subsystem exposes the `addVisionMeasurement` method to enable vision systems to publish samples. Additional methods can be easily exposed as desired.

Alternatively, other pose estimation systems can be easily integrated in place of the WPILib solution. The `periodic` method includes a call to `poseEstimator.updateWithTime` that includes the sample timestamp, gyro rotation, and module positions. This call can be replaced to integrate with any other odometry or pose estimation system.

### Swerve Setpoint Generator

The project already includes basic mechanisms to reduce skidding, such as drive current limits and cosine optimization. Users who prefer more control over module skidding may wish to utilize Team 254's [`SwerveSetpointGenerator`](https://github.com/Team254/FRC-2023-Public/blob/main/src/main/java/com/team254/lib/swerve/SwerveSetpointGenerator.java) in the `runSetpoint` method of the `Drive` subsystem.

### Advanced Physics Simulation

The project can be easily adapted to utilize Team 5516's [maple-sim](https://github.com/Shenzhen-Robotics-Alliance/Maple-Sim) library for simulation, which provides a full rigid-body simulation of the swerve drive and its interactions with the field. Check the documentation for more details on how to install and use the library.