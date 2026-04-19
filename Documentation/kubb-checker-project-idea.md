# Kubb-Checker Project

Kubb-Checker shall assist the referee in a game of Kubb to decide wether a throw was valid or not.

## Main Function

Kubb-Checker consists of two parts: 

1. Kubb-checker-sensor:  The first part, the "Kubb-checker-sensor", is an embedded device consisting of an ESP32-C3 with a BMI160 gyroscope connected via I2C bus. The Kubb-chcker-sensor is placed inside a wooden baton (ca. 30cm long and ca. 4cm in diameter). There are at least six kubb-checker-sensors per game. As part of the game, the baton is thrown at the kubbar (wooden blocks). The rules of the game require that the baton has turned at least 360° around his minor axis between leaving the hand of the thrower and hitting a kubbar. The embedded SW shall querry the gyroscope and inferr from the gyro sensor data if and when the baton was thrown, when it hits the kubbar (or any other target). During this period the baton is assumed to be flying. After detecting the end of a flight, the rotation in degrees around the major and minor axis of the baton and the time-of-flight shall be computed.

2. Kubb-checker-hub: The "baton-sensor" is connected via ESP-now to the second part of the system, the "Kubb-checker-hub". The hub is the center of a one-to-many ESP-now communication (star topology). At the same time, the hub opens an WiFi accesspoint to allow mobile devices to connect to the embedded web-server. The hub collects and stores all data about throws from all connected baton-sensors. The data consist at least of a state (still, moving, throw) and the information about the last detected throw (time-of-day of the start of the throw, time-of-day of the end of the throw, measured rotation around the minor axis, measured rotation around the major axis, maximum longitudinal accelleration). The data shall be presented on a graphical web interface via WiFi to all mobile devices connected to the embedded WiFi accesspoint.
The web interface presents the data of each connected kubb-checker-sensor and also accumulates additional statistics during the game (per baton: the total number of throws; the maximal, minimal, and average rotation; the maximal, minimal, average accelleration; the maximal, minimal, average time-of-flight). For each throw it shall be indicated if the throw was valid or not. A throw is valid if the rotation around the minor axis during the flight was more tha 360°.

## Other Functions

All ESPs shall support OTA to be flashed wirelessly.
All kubb-checker-sensor devices shall output all info which is sent to the hub also via the serial debuggin interface.
All kubb-checker-sensor devices shall be given an unique name (text) by the user via the web-interface
All kubb-checker-sensor devices shall calibrate the gyroscope at start-up. A manual calibration function shall be accessible via the web interface
The kubb-checker-sensor shall monitor the state-of-charge ot the attached LiIon cell (1S) and send it to the hub. The hub shall present the current state of charge of all attached sensors via the web interface.
The game-statistics shall be resetted via a button in the web-interface.

## Implementation Phases

1. Implement all functions of the kubb-checker-sensor except the connection to the hub. The information shall be monitored and debugged via the serial interface connected to the sensor device.
2. Implement all functions of teh hub exceot the web interface. The information shall be monitored and debugged via the serial interface connected to the hub device.
3. Implement the web interface.

## Tools

