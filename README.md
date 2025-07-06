# Readme

## Overview 
this file explains Full_Potential_Steering.py file 

## ROS Message Types Used

### `sensor_msgs/Joy` Message

Reports the state of joystick axes and buttons.  
**Message structure:**

```
Header  header         
float32[] axes         # the axes measurements from a joystick  
int32[] buttons        # the buttons measurements from a joystick
```

**Example:**

```
Header: 
  stamp: 
    secs: 1722450542
    nsecs: 123456789
Axes: [0.0, -1.0, 0.5, 0.0]       # e.g., left stick Y fully down, right stick X halfway  
Buttons: [0, 1, 0, 1, 0, 0]       # e.g., Button 1 and 3 are pressed
```

---

### `std_msgs/Int32MultiArray`

Used to send PWM data.
Structure of `pwm_msg`:

```
[fld, frd, bld, brd, fls, frs, bls, brs]
```

* `fld = front left drive` (all integer values)

---

### `std_msgs/Float32MultiArray`

Used to receive encoder data from an external node.

---

## Joystick Button and Axis Mapping

| Name                         | Purpose                             |
| ---------------------------- | ----------------------------------- |
| `steer_unlock_axis`          | Pressing it puts in configuration 2 |
| `full_potential_unlock_axis` | Pressing it puts in configuration 3 |

> Full potential is locked if steer is unlocked and vice versa.

---

### Configuration 1

| Name                | Purpose                              |
| ------------------- | ------------------------------------ |
| `forward_btn`       | Turn all wheels to face forward      |
| `parallel_btn`      | Turn all wheels by 90° clockwise     |
| `rotinplace_btn`    | Rotate rover in place (crab)         |
| `fb_axis`           | Control forward/backward motion      |
| `lr_axis`           | Control left/right motion            |
| `rot_with_pwm`      | Manually rotate using PWM (in-place) |
| `state_ctrl_button` | Change to joystick/autonomous mode   |

![Configuration 1](Pasted%20image%2020250706143540.png)

---

### Configuration 2

| Name                 | Purpose                                                      |
| -------------------- | ------------------------------------------------------------ |
| `forward_btn`        | Steer all wheels clockwise to 45° (forward direction)        |
| `parallel_btn`       | Steer all wheels anticlockwise to 45° (sideways direction)   |
| `steer_samedir_axis` | Manually control all steering wheels in same direction (PWM) |
| `steer_oppdir_axis`  | Rotate front wheels clockwise, back wheels anticlockwise     |

![Configuration 2](Pasted%20image%2020250706143437.png)

---

### Configuration 3

| Name            | Purpose                                     |
| --------------- | ------------------------------------------- |
| `fl_wheel_axis` | Manually control front-left steering motor  |
| `fr_wheel_axis` | Manually control front-right steering motor |
| `bl_wheel_axis` | Manually control back-left steering motor   |
| `br_wheel_axis` | Manually control back-right steering motor  |

![Configuration 3](Pasted%20image%2020250706143558.png)

---

## `joyCallback`

### `self.mode`

Integer variable storing current **drive mode** level (`0` to `3`):

* Press `modeupbtn` → `self.mode`++
* Press `modednbtn` → `self.mode`--

### `self.state`

Boolean variable to toggle between **joystick** and **autonomous** mode:

* If `self.state == False` → run `steering()` → `drive()`
* If `self.state == True` → run `drive()` only

---

### Configuration 1: both locked

* `steering_ctrl_locked`: state of `forward_btn`, `parallel_btn`, `rotinplace_btn`
* `drive_ctrl`: joystick axis values for `fb_axis` and `lr_axis`
* `rot_with_pwm`: axis value to enable manual rotation using PWM

---

### Configuration 2: steer unlocked

* `steering_ctrl_unlocked`: state of `forward_btn`, `parallel_btn`
* `steering_ctrl_pwm`: values of `steer_samedir_axis` and `steer_oppdir_axis` for manual PWM steering

---

### Configuration 3: full potential unlocked

* `full_potential_pwm`: axis values = `[fl_wheel_axis, fr_wheel_axis, bl_wheel_axis, br_wheel_axis]`

---

## `EncCallback`

Encoder data is received as `Float32MultiArray` from another node.
It is stored in `enc_data` as:

| Index | Wheel                |
| ----- | -------------------- |
| 0     | Front Left encoder   |
| 1     | **-1 × Front Right** |
| 2     | **-1 × Back Left**   |
| 3     | Back Right encoder   |

> Negation ensures consistent encoder polarity.

---

## `Steering()`

### `Steer()`

**Attributes:**

* `initial_angles`, `final_angles`, `mode`

| Mode | Description       |
| ---- | ----------------- |
| 0    | Relative movement |
| 1    | Absolute movement |

Steers until all wheels are within a threshold (or timeout).
PWM is computed using a proportional controller (`kp_steer`) and published.

```python
self.pwm_msg.data = [
    0, 0, 0, 0,                    # Drive motors (not used here)
    pwm[0] * self.init_dir[4],    # Front Left Steering
    pwm[1] * self.init_dir[5],    # Front Right Steering
    pwm[2] * self.init_dir[6],    # Back Left Steering
    pwm[3] * self.init_dir[7]     # Back Right Steering
]
```

> `init_dir` is used to flip motor directions if needed.

---

### Configuration 1: both locked

* `forward_btn` → call `steer()` with `final_angle = initial_angle`
* `parallel_btn` → `final_angles = [90, 90, 90, 90]`
* `rotinplace_btn` → `final_angles = [45, -45, -45, 45]` and set `rotinplace = True`
* `rot_with_pwm` (axis) → generate manual PWM, no `steer()` call

---

### Configuration 2: steer unlocked

> `_enc_data_new = copy.deepcopy(self.enc_data)`
> Used to ensure deep copy (not pointer) for `initial_angles`.

* `forward_btn` → steer to `+45°`
* `parallel_btn` → steer to `-45°`
* `steer_same_dir_axis` → same PWM to all wheels (no `steer()` call)
* `steer_opp_dir_axis` → front clockwise, back anticlockwise

---

### Configuration 3: full potential unlocked

`full_potential_pwm = [fl, fr, bl, br]`
Each axis directly controls corresponding steering motor via PWM.

---

## `drive()`

> Only active in **Configuration 1** (both locked)

* If `rotinplace_btn` →

  1. Use `steering()` to move into crab position
  2. Use `drive()` to publish PWM for drive motors:

     ```python
     vel = self.d_arr[self.mode] * self.drive_ctrl[1]
     ```

---

### If `self.state == False` (joystick mode)

* `velocity = -self.d_arr[self.mode] * self.drive_ctrl[1]`
* `omega = -self.d_arr[self.mode] * self.drive_ctrl[0]`

These are smoothed using a **moving average queue** for stability.
PWM is then published using differential drive logic.

---

### If `self.state == True` (autonomous mode)

Use predefined autonomous `velocity` and `omega`, and publish PWM in same differential manner.

---
