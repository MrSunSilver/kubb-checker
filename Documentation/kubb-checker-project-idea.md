# Kubb-Checker Project

Kubb-Checker shall assist the referee in a game of Kubb to decide wether a throw was valid or not. A throw is valid if the baton is rotating vertically (within some margin), not horizontally (like a helicopter) and the baton has rotated at least one full rotation. 

## Main Function

Kubb-Checker consists of two parts: 

### baton
The first part, the "Kubb-checker-sensor", is an embedded device consisting of an ESP32-C3 with a BMI160 gyroscope connected via I2C bus. The Kubb-chcker-sensor is placed inside a wooden baton (ca. 30cm long and ca. 4cm in diameter) relatively close to the center of gravity. There are at least six kubb-checker-sensors per game. As part of the game, the baton is thrown at the kubbar (wooden blocks). The rules of the game require that the baton has turned at least 360° around his minor axis between leaving the hand of the thrower and hitting a kubbar. The embedded SW shall querry the gyroscope and inferr from the gyro sensor data if and when the baton was thrown, when it hits the kubbar (or any other target). During this period the baton is assumed to be flying. After detecting the end of a flight, the rotation in degrees around the major and minor axis of the baton and the time-of-flight shall be computed.

#### BLE-connection
The Kubb-checker-sensor is paired via BLE to a smart phone. For now only Android phones are supported. The pairing will be facilitated via a 215 NFC tag, attached to the baton. To wright the NFCtag and prepare tap-to-pair OOB, the BLE-MAC-address is needed. The BLE-MAC-addresss shall be logged together with the default baton-ID ("kastpinne-xxxx" where the xxxx are replaced by the last four hexadecimal digits of the BLE-MAC) to serial console. connection status and some debugging info shall also be logged to serial console. After pairing, the Kubb-checker-sensor shall transmit data to the phone app via BLE:
Event-Characteristic (NOTIFY): The main data about the "throw"-event. now data must be lost.
   event-counter;
   time-of-day of the event;
   rotation around major and minor axes in degrees;
   time-of-flight in s;
   hight and distance of flight in m;
   the maximal, minimal, average translation accelleration in m/s²; 
   the maximal, minimal, average rotation accelleration in degrees/s²; 

Telemetrie-Characteristic (NOTIFY): continuous background stream with reduced data rate e.g. 10Hz with gyro raw data (rotations and accellerations) and some derived signals for live display in the app. some samples may be lost. 

Status-Characteristic (READ + NOTIFY): battery, uptime, status of calibration, firmware version; rarely changed but sent upong request by the app

Control-Characteristic (WRITE) — App → ESP: for commands like: "calibration", "set name", "start", "stop", "change thresholds".  the set name command changes the name of the device permanently, persisting over reboots. same for the calibration and threshold data, it shouldnotget lost between reboots ot resets.

#### Gyro-calibration
This function can be triggered via a BLE command from the Kubb-checker-App. The App will request the user to make sure the baton is lying still on a flat surface. A sanity check shall be performed to asure, that the baton apears to be relatively at rest, othewise an error should be reported back to the app. If the the sanity check passes, it is safe to asume that the baton is lying on a flat surface and the gyro should neither register rotation nor accelleration other than gravity. After some averaging of measured drifts, some calibration parameters can be derived to acount for in all other calculations later. The calibration parameters should be stored to persist after rests and reboots.

#### throw-detection

#### shake-detection

### Kubb-checker-App
This is an app on a smart phone. The "baton-sensors" are connected via BLE to the second part of the system, the "Kubb-checker-App". The App is the user interface for connecting to the batons, calibrating the batons, collecting and displaying game data from the batons.

#### front-page
At the top of the front-page the app-name "Kubb Checker" shall be written centered. The rest of the page is devided into six equal framed areas. each area dedicated to on baton. If less than six batons are connected the remaining areas are empty.
Each area shall display a sumamry of the data from the batons relevant to the referee of the game, i.e. the baton-id, the statistics of the last throw and an indicator if the baton is moving or resting still. Most important, it shall be indicated with two indicators (red/green) if the last throw was valid with respect to the two criteria: (1) rotating vertically not horizontally (2) at least on full rotation. The are of the baton which was thrown last shall be indicated with a nice highlighted frame.

On the upper right of the page, next to the title, but on the very right, shall be a button with a gears icon. when pressed the sewttings-sub-page page shall be opened.

#### settings-sub-page
On the "Settings" page the app shall display a list of all paired Batons with MAC and name and an indicator for shaking / rapid movement to identify individual batons by the user. when the user selects a baton, the baton-sub-sub-page shall show the baton-id followed by a 3D model of a wooden cylinder with slightly rounded edges, representing the baton and rotating acording to the gyro readings from this specific baton. Three bar-graphs shall indicate the accellerations in each degree of translational freedom on a scale between 0 and 100m/s². lower on the page shall be a button to start a gyro-calibration. This procedure is described in section Gyro-calibration.
Next to the baton-id shall be a small "edit" button to change the baton-ID. when the button is pressed, the text-field with the batton-ID shal change into a input field. after input has finished, the baton-id shall be written to the baton via BLE control-characteristic.

## app-design
The app shall have a modern slick neumorphic design with light gray and white and shades of green resembling the grass of the kubb field.

## Other Functions
All ESPs shall support OTA to be flashed wirelessly.
All kubb-checker-sensor devices shall log relevant debugging inforamtion to the serial interface.

## Implementation Phases

### phase-1 
focus on BLE connection
kubb-checker-sensor: basic, just supporting BLE pairing to app, constantly sending demo-generated (simple rotation around one axis with some constant accelleration in a random directon) Telemetrie-Characteristic data to the app via BLE.
kubb-checker-app: supporting BLE pairing and connection. Display of the Telemetrie-Characteristic just as text from just one baton.

### phase-2
focus on gyro
kubb-checker-sensor: read data via I2C fro the gyro and send it via Telemetrie-Characteristic to the app

### phase-3
focus on calibration
kubb-checker-sensor: implement the calibration functionality upon request via Control-Characteristic.
kubb-checker-app: implement the settings-sub-page, but with just one list entry for the one baton yet. implement the baton-sub-sub-page showing telemetry-data from the baton with the 3D-Model and featuring the calibration button to request the calibration routine frrom the baton.

### phase-4
focus on throw-event
kubb-checker-sensor: analyse the gyro-signals and detect the start and end of a throw. The start is probably characterized by a phase of medium accelleration, both translation and rotation, followed by a phase of free flight with just gravity and then a relatively harsh impact and finally, after some more sliding or tumbling, a stop of any motion. If the baton hit a wooden kubb the impact might my harsher than if it landed on grass.
the phse of free flight is what is of interest.  Analyze how long it was in time (s) and how many rotations around which axis happened. in which direction did the minor-rotational-axis point? a valid throw (vertical rotation) means the axis is pointing roughly anywhere in he direction of the horizon, good. if the minor rotational axis was pointing up or down, it indicates a helicopter throw, invalid.
implement the Event-Characteristic including transmission to the app.
kubb-checker-app: on the game-page display all data from the Event-characteristic with some nice intuitive and UI elements to condense the information.

### phase-5
focus on kubb-checker-sensor feature-completeness
kubb-checker-sensor: implement collecting and sending Status-Characteristic and all remaining features
kubb-checker-app: implement the UI for the Status-Characteristic in the baton-sub-sub-page

### phase-6
multiple batons
kubb-checker-sensor: probably not much to be done here?
kubb-checker-app: implement the full support for up to six connected batons, both on the game-page and in the settings-sub-page.

## Tools

