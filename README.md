# Task 6 – Red Light Green Light Game (Squid Game) Using ESP32-CAM, OpenCV, and Buzzer

## 1. Aim

To develop a motion detection game inspired by *Squid Game: Red Light, Green Light* using an ESP32-CAM, OpenCV-based image processing, and a buzzer to detect player movement during the red-light phase and generate an elimination sound.

## 2. Introduction

Red Light, Green Light is a game in which players move during the green-light phase and must remain still during the red-light phase. A player who moves during the red-light phase is considered eliminated.

In this project, an ESP32-CAM captures video of the player and provides image data to a computer running Python and OpenCV. OpenCV processes the video frames to detect movement by comparing consecutive images. The game alternates between red-light and green-light phases. If movement is detected during the red-light phase, the system activates a buzzer to indicate elimination.

This project demonstrates the practical application of embedded systems, computer vision, image processing, and sound-based feedback in an interactive game.

## 3. Components Required

1. ESP32-CAM module
2. USB-to-serial programmer or compatible ESP32-CAM programmer
3. Active buzzer
5. Jumper wires
6. Stable power supply
7. Computer or laptop
8. Python programming environment
9. OpenCV library
10. Wi-Fi connection

## 4. System Architecture

The system consists of three main sections:

- **Image acquisition:** The ESP32-CAM captures video frames of the player.
- **Motion detection and game logic:** Python and OpenCV process the video frames and determine whether movement occurs during the red-light phase.
- **Alert generation:** The computer sends a control signal to the ESP32, which activates the buzzer when a player is eliminated.

### Block Diagram

```text
        Player
          |
          v
      ESP32-CAM
          |
          | Video stream over Wi-Fi
          v
   Computer / Laptop
          |
          v
    Python + OpenCV
          |
          v
    Motion Detection
          |
          v
    Game Phase Check
          |
          v
 Movement during RED?
       /       \
     YES        NO
      |          |
      v          v
  Elimination   Continue
      |
      v
 Send command to ESP32
      |
      v
    Buzzer ON
```

## 5. Circuit Connections

### A. ESP32-CAM to Buzzer

For a small buzzer compatible with the selected GPIO output, a direct connection may be possible if the electrical specifications permit it. Otherwise, use a transistor driver.

| Buzzer / Driver Connection | ESP32-CAM Connection |
|---|---|
| Driver control input | GPIO 13 |
| Driver GND | GND |
| Buzzer positive | Suitable external supply, as required |
| Buzzer negative | Driver output |

**Important:** Do not connect a high-current buzzer directly to an ESP32 GPIO pin. Use a transistor or MOSFET driver and an appropriate power supply. Verify that GPIO 13 is available on your specific ESP32-CAM board.

### B. ESP32-CAM Power

Supply the ESP32-CAM using a stable power source suitable for the board. Ensure the power supply can handle current peaks from Wi-Fi and camera operation.

## 6. Working Principle

1. The ESP32-CAM captures video of the player.
2. The video stream is accessed by a computer over Wi-Fi.
3. Python and OpenCV process the incoming frames.
4. The system compares consecutive frames to estimate whether the player is moving.
5. During the green-light phase, movement is permitted.
6. During the red-light phase, the player must remain still.
7. If movement exceeds the selected threshold during the red-light phase, the player is marked as eliminated.
8. The computer sends a command to the ESP32-CAM.
9. The ESP32-CAM activates the buzzer to produce an elimination sound.

## 7. Software Requirements

Install Python on the computer and install the required packages using a terminal:

```bash
pip install opencv-python requests
```

The ESP32-CAM firmware must provide a video stream and accept a command to control the buzzer. The following Python program assumes the camera stream is available at a URL such as `http://192.168.1.100/stream` and that the ESP32 firmware accepts a buzzer command at `http://192.168.1.100/buzzer?state=on`.

Replace the example IP address and endpoint paths with those actually configured in your ESP32 firmware.

## 8. Python Code for Motion Detection

The following example implements the basic game logic using frame differencing. It assumes that the ESP32-CAM is already streaming video and that a compatible buzzer-control HTTP endpoint is available.

```python
import cv2
import time
import requests

CAMERA_URL = "http://192.168.1.100/stream"
BUZZER_URL = "http://192.168.1.100/buzzer"

RED_LIGHT_DURATION = 8
GREEN_LIGHT_DURATION = 8
MOTION_THRESHOLD = 1800

cap = cv2.VideoCapture(CAMERA_URL)

if not cap.isOpened():
    raise RuntimeError("Could not open ESP32-CAM stream")

previous_gray = None
phase = "GREEN"
phase_start = time.time()
eliminated = False

def set_buzzer(state):
    try:
        requests.get(
            BUZZER_URL,
            params={"state": state},
            timeout=1
        )
    except requests.RequestException:
        print("Could not contact buzzer controller")

while True:
    ret, frame = cap.read()

    if not ret:
        print("Failed to read camera frame")
        break

    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    gray = cv2.GaussianBlur(gray, (21, 21), 0)

    motion_pixels = 0

    if previous_gray is not None:
        diff = cv2.absdiff(previous_gray, gray)
        _, thresh = cv2.threshold(
            diff, 25, 255, cv2.THRESH_BINARY
        )
        motion_pixels = cv2.countNonZero(thresh)

    previous_gray = gray.copy()

    elapsed = time.time() - phase_start

    if not eliminated:
        if phase == "GREEN" and elapsed >= GREEN_LIGHT_DURATION:
            phase = "RED"
            phase_start = time.time()
            previous_gray = None

        elif phase == "RED" and elapsed >= RED_LIGHT_DURATION:
            phase = "GREEN"
            phase_start = time.time()
            previous_gray = None

        if phase == "RED" and motion_pixels > MOTION_THRESHOLD:
            eliminated = True
            set_buzzer("on")
            print("PLAYER ELIMINATED!")

    color = (0, 255, 0) if phase == "GREEN" else (0, 0, 255)

    cv2.putText(
        frame,
        f"LIGHT: {phase}",
        (20, 40),
        cv2.FONT_HERSHEY_SIMPLEX,
        1,
        color,
        2
    )

    if eliminated:
        cv2.putText(
            frame,
            "ELIMINATED!",
            (20, 90),
            cv2.FONT_HERSHEY_SIMPLEX,
            1,
            (0, 0, 255),
            3
        )

    cv2.imshow("Red Light Green Light", frame)

    key = cv2.waitKey(1) & 0xFF

    if key == ord("q"):
        break

    if key == ord("r"):
        eliminated = False
        phase = "GREEN"
        phase_start = time.time()
        previous_gray = None
        set_buzzer("off")

cap.release()
cv2.destroyAllWindows()
set_buzzer("off")
```

### Important Notes About the Python Code

- The example uses frame differencing, a basic computer-vision technique, rather than a trained image recognition model.
- `MOTION_THRESHOLD` must be calibrated for the camera resolution, player distance, lighting, and background.
- The example does not include ESP32-CAM firmware for the stream or buzzer endpoint. Those endpoints must be implemented separately.
- Changes in lighting, camera movement, shadows, and background activity can trigger false detections.
- For improved player-specific detection, a pose estimation or person tracking model can be added.


## 9. Advantages

1. Provides an interactive motion detection game.
2. Demonstrates real-time video processing.
3. Combines embedded hardware with computer vision.
4. Uses sound to provide immediate feedback.
5. Helps develop practical skills in Python, OpenCV, and ESP32 programming.

## 10. Applications

- Interactive gaming projects
- Computer-vision demonstrations
- Motion detection experiments
- Educational robotics
- Real-time movement monitoring prototypes
