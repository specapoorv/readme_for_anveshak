#### Subscribers

`/state` gives data in form of `Bool`  
zed2i's imu gives data as `IMU` message  
`enc_arm` gives data as `Float32MultiArray`  
`/gps_coordinates` gives data as `NavSatFix`  
zed2i's rgb image comes as `Image`  
zed2i's depth image comes as `Image`

#### Publishers

publishing `Int32MultiArray` on `stm_write`  
publishing `WheelRpm` on `/motion`  
publishing `Int8` on `/gps_bool`  
publishing `Int8` on `/rot`

#### cv bridge

used to convert ros image messages to opencv images  
then hsv range mask is applied from `[0, 150, 20]` to `[15, 255, 255]`

#### templates

left and right arrow images are stored  
gaussian blur applied  
these are used as templates in template matching

---

#### callbacks

`color_callback`  
subscriber = zed2i rgb  
convert ros image to opencv using `bgr8` encoding  
crop to center and store in `self.color_image`  
set flag `self.got_image = True`

`depth_callback`  
subscriber = zed2i depth  
convert ros image to opencv using `passthrough` encoding  
crop to center and store in `self.depth_image`

`gps_callback`  
subscriber = `/gps_coordinates`  
check if `msg.latitude` and `msg.longitude` exist  
store in `self.current_latitude` and `self.current_longitude`

`enc_callback`  
subscriber = `enc_arm`  
store second value from `Float32MultiArray` into `self.encoder_data`

`yaw_callback`  
take `imu.orientation` and convert quaternion to euler  
take `z` angle and store in `self.z_angle_in_rad`  
on first message store `self.initial_yaw = self.z_angle_in_rad`  
later on update `self.z_angle` by wrapping the angle in `[-π, π]`

---

#### key methods

`get_box`  
runs yolov5 on `self.color_image` using model `best_cube.pt`  
confidence threshold = 0.5, max 2 detections  
extracts bbox: `latest_xmin`, `latest_ymin`, `latest_xmax`, `latest_ymax`  
sets `ret = True` if any detection else `ret = False`  
shows image using cv2.imshow

`cone_model`  
runs yolov5 using cone model  
confidence threshold = 0.7  
checks depth at center of detected object  
returns `ret`, detection type string, center coords, depth

`arrowdetectmorethan3`  
runs when distance > 1.5  
uses yolov5 detection  
checks if `color_image`, `depth_image` and `results` exist  
takes center of arrow bbox  
reads corresponding depth  
returns `ret`, direction as string, center coords, depth

`arrowdetectlessthan3`  
runs when distance < 1.5  
applies gaussian blur and grayscale  
template match left and right templates  
multi-scale match from 0.03 to 0.3  
best match picked using `TM_CCOEFF_NORMED`  
threshold = 0.71  
if match valid, draw rectangle, read depth at center  
returns `ret`, direction as "left"/"right", center, depth

`cone`  
convert `color_image` to HSV  
apply HSV mask in red/orange range  
morphological ops + median blur  
find contours, compute convex hulls  
filter by area and aspect ratio  
check for upward-pointing shapes  
returns list of valid cone bboxes

`move_straight`  
publishes `WheelRpm` for forward motion  
uses proportional control for velocity based on `circle_dist`  
if within `circle_dist`, stop rover  
average 100 GPS readings and save to `coordinates.txt` and `sparsh2.csv`  
set `turn = True` after gps logging  
calls `v1_competition` if competition over

`v1_competition`  
stop rover  
keep checking `/state` until it becomes `True` (joystick mode)  
print status messages  
shutdown after that

`process_dict`  
look for min distance key in `angles_dict`  
average encoder values for that key  
decide direction `left` or `right`  
compute `rotate_angle` from encoder  
set `turn = True`  
reset `angles_dict`

`rotate`  
use `imu` yaw to rotate rover  
compute `error = rotate_angle - abs(diff)`  
publish `WheelRpm` with angular velocity depending on direction  
stop rotation when error < `0.5 * angle_thresh`  
store `initial_drift_angle` and reset variables

`rot_in_place`  
used for close arrows  
similar to rotate but `omega = 40`  
publish `rot` to toggle steering  
increment `count_arrow`  
reset internal flags

`search`  
rotate arm to detect arrows  
loop for 20 seconds or until arrow is found  
sweep from `start_angle = 50` back to center  
store encoder angle and distances in `angles_dict`  
after search, call `process_dict`  
reset arm to center

`quaternion_to_euler`  
standard conversion using `roll`, `pitch`, `yaw` formulas from quaternion

`main`  
runs navigation logic  
if `count_arrow < arrow_numbers`:  
→ if `ret = False` or `init = True`, call `search`  
→ if search done, call `process_dict`  
→ detect arrows based on distance > 1.5 or < 1.5  
→ use `arrowdetectmorethan3` or `arrowdetectlessthan3`  
→ based on result, call `move_straight`, `rotate`, or `rot_in_place`

if `count_arrow == arrow_numbers`:  
→ run `cone_model`  
→ use `move_straight` or `rotate` depending on cone position  
→ call `v1_competition` after course is done

`run`  
main loop  
rospy rate = 10Hz  
if `image_avbl = True` and `count_arrow < arrow_numbers` then call `get_box`  
then call `main`  
handle ros shutdown and close cv2 windows

---

#### control flow

init:  
call `rospy.init_node('zed_depth')`  
set publishers and subscribers  
load YOLO models and arrow templates

main loop (`run`):  
runs at 10Hz  
checks image availability  
calls `get_box`  
calls `main`

`main`:  
if `count_arrow < arrow_numbers`  
→ call `search` if no arrow found or if in init mode  
→ process search result  
→ detect arrow (near or far)  
→ navigate straight or rotate

if `count_arrow == arrow_numbers`  
→ detect cones  
→ navigate accordingly  
→ end course with `v1_competition`

`search`:  
rotate arm and check for arrow  
collect encoder angle and distance  
store in `angles_dict`  
process that to decide rotate angle

`move_straight`, `rotate`, `rot_in_place`:  
control rover’s motors  
adjust based on depth/yaw/GPS  
log coordinates and move forward  
handle in-place turns

callbacks:  
update images, imu, gps, encoder continuously

end:  
once arrows and cones are done  
stop rover  
wait for joystick mode  
shutdown
