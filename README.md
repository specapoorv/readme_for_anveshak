# readme_for_anveshak
#### ROS Message Types Used
*sensor_msgs/Joy Message* : reports the state of joystick axes and buttons, 
msg structure is this -> 
Header header         
float32[] axes          # the axes measurements from a joystick  
int32[] buttons         # the buttons measurements from a joystick

example : 
	Header: 
		  stamp: 
		    secs: 1722450542
		    nsecs: 123456789
	Axes: [0.0, -1.0, 0.5, 0.0]       # e.g., left stick Y fully down, right stick X halfway
	Buttons: [0, 1, 0, 1, 0, 0]       # e.g., Button 1 and 3 are pressed


*std_msgs/Int32MultiArray* : we use this msg to send pwm data 
structure of pwm_msg is like this -> [fld, frd, bld, brd, fls, frs, bls, brs]
(fld = front left drive and all integer values)

_std_msgs/Float32MultiArray_ : we use this msg to receive encoder data from an external node 

#### Joystick Button and Axis Mapping

| Name                         | Purpose                             |
| ---------------------------- | ----------------------------------- |
| `steer_unlock_axis`          | pressing it puts in configuration 2 |
| `full_potential_unlock_axis` | pressing it puts in configuration 3 |
also we will lock full potential if steer is unlocked and vice versa
###### Configuration 1

| Name                | Purpose                                 |
| ------------------- | --------------------------------------- |
| `forward_btn`       | Turn all wheels to face forward         |
| `parallel_btn`      | Turn all wheels by 90 degrees clockwise |
| `rotinplace_btn`    | Rotate rover in place (crab)            |
| `fb_axis`           | Control forward/backward motion         |
| `lr_axis`           | Control left/right motion               |
| `rot_with_pwm`      | Manually rotate using PWM (in-place)    |
| `state_ctrl_button` | change to joystick/autonomous           |
![[Pasted image 20250706143540.png]]
###### Configuration 2

| Name                 | Purpose                                                      |
| -------------------- | ------------------------------------------------------------ |
| `forward_btn`        | Steer all wheels clockwise to 45° (forward direction)        |
| `parallel_btn`       | Steer all wheels anticlockwise to 45° (sideways direction)   |
| `steer_samedir_axis` | Manually control all steering wheels in same direction (PWM) |
| `steer_oppdir_axis`  | Rotate front wheels clockwise, back wheels anticlockwise     |
![[Pasted image 20250706143437.png]]
###### Configuration 3

| Name            | Purpose                                     |
| --------------- | ------------------------------------------- |
| `fl_wheel_axis` | Manually control front-left steering motor  |
| `fr_wheel_axis` | Manually control front-right steering motor |
| `bl_wheel_axis` | Manually control back-left steering motor   |
| `br_wheel_axis` | Manually control back-right steering motor  |
![[Pasted image 20250706143558.png]]
##### joyCallback :
- `self.mode`:  
    An integer variable that stores the current **drive mode** level (ranging from `0` to `3`).
    - Pressing `modeupbtn` increments `self.mode` (up to a maximum of 4 total modes)
    - Pressing `modednbtn` decrements `self.mode` (down to 0)
- `self.state`:  
    A boolean variable used to toggle between  _joystick control_ and *autonomous mode*, toggled by pressing `state_ctrl_btn`.
    if `self.state` == False | run steering() -> drive ()
    if `self.state` == True | run drive()
###### configuration 1 : both not pressed (both locked)
- `steering_ctrl_locked`: stores the state of `forward_btn`, `parallel_btn`, and `rotinplace_btn`
- `drive_ctrl`: stores joystick axis values for `fb_axis` and `lr_axis` 
- `rot_with_pwm`: stores the value of `rot_with_pwm` axis to enable manual rotation using PWM
###### configuration 2 : steer_unlock_axis is pressed (steer unlocked)
-  `steering_ctrl_unlocked`: stores the state of `forward_btn` and `parallel_btn`
-  `steering_ctrl_pwm`: stores the values of `steer_samedir_axis` and `steer_oppdir_axis` for manual PWM-based steering control
###### configuration : 3 full potential unlocked
-  `full_potential_pwm`: stores the individual axis values for controlling each wheel's steering independently == `fl_wheel_axis`, `fr_wheel_axis`, `bl_wheel_axis`, `br_wheel_axis`

##### EncCallback 
we are getting enc data from another script in float32 values 
we initialise an array enc_data and then store the data in this order 
index :
	0 : front left wheel encoder
	1: negative of front right
	2: negative of back left
	3: back right wheel encoder
negation ensures consistent encoeder polarity

#### Steering() 
##### Steer() 
--- attributes : `initial_angles`, `final_angles`, `mode`

`mode` == 0 :  relative movement -> steer each wheel to go final angle away from initial_angle
`mode` == 1 : absolute movement -> steer each wheel by absolute angles
it steers until all 4 wheels are inside of an error threshold (also breaks if exceeds time threshold) 
pwm is computed using proportional controller kp_steer 
and pwm_msg is published
then at the end puts steering_complete flag as True

self.pwm_msg.data = [
    0, 0, 0, 0,                    # Drive motors (not used here)
    pwm[0] * self.init_dir[4],    # Front Left Steering
    pwm[1] * self.init_dir[5],    # Front Right Steering
    pwm[2] * self.init_dir[6],    # Back Left Steering
    pwm[3] * self.init_dir[7]     # Back Right Steering
]

we use `self.init_dir` to flip control direction if there was some motor wiring or mounting direction issue 
###### Configuration 1 : both locked
- if `forward_btn` is pressed -> call `steer()` but `final angle = initial angle`
- if `parallel_btn` is pressed → call `steer()` with `final_angles = [90, 90, 90, 90]` 
- if `rotinplace_btn` is pressed → call `steer()` with `final_angles = [45, -45, -45, 45]` and set `rotinplace = True` (rotation-in-place setup)
- if `rot_with_pwm` axis is moved (instead of a button) → manually generate PWM for crab-style in-place rotation, without calling `steer()`.
###### configuration 2 : steer unlocked
_doubt how deepcopy works             enc_data_new = copy.deepcopy(self.enc_data) # to create a deep copy of enc_data array, not a pointer equivalence. Is being used as the inital angle_
- if `forward_btn` is pressed → steer all wheels **clockwise** to `+45°`
- if `parallel_btn` is pressed → steer all wheels **anticlockwise** to `-45°`
- if `steer_same_dir_axis` is moved → manually control all wheels with the same PWM (as in `forward_btn` case, but without calling `steer()`)
- if `steer_opp_dir_axis` is moved → front wheels rotate clockwise, back wheels rotate anticlockwise, enabling a "twist" motion
###### Configuration 3 : full_potential unlocked 
`full_potential_pwm` = [fl,fr,bl,br]
each axis controls each steering motor and we can manually give pwm input
#### drive()
only works when both steering and full_potential is locked i.e __configuration 1__

- if rotinplace_btn pressed it will first go through `steering()` function to turn rover
	into crab position and then it goes in `drive()` where it publishes pwm_msg for *drive motors* with pwm that can be chosen using _self.mode_ (0 to 3) we can choose  from this array -> self.d_arr = [35,50,75,110,150] and then we can  finetune the pwm using lr_axis 
	vel = self.d_arr[self.mode] * self.drive_ctrl[1]

-  if `self.state == False` that is joystick control
	velocity is calculated using lr_axis and omega is calculated using fb_axis
	`velocity` = -self.d_arr[self.mode] * self.drive_ctrl[1] # lr axis
    `omega` = -self.d_arr[self.mode] * self.drive_ctrl[0] # fb axis
    then a moving average of velocity and omega is calculated by putting the values in queue, we do this because joystick axis might not be stable enough to give values 

	then pwm_msg is published in differential drive manner using avg velocity and avg omega

- if `self.state == True` that is autonomous mode
	then a predefined autonomous velocity and omega is given and again in differential drive manners its published
	
