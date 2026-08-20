## 👀 run-cv-model-in-rp5 Overview  3/3  
<h1 align="center">Run computer vision model in Raspberry Pi 5 AI HAT</h1>  


## 🔎 Preparation
Open your rasberry pi and follow these steps:
1. `sudo raspi-config`
2. `Advanced Options → PCIe Speed → Yes / Enable`
3. `Interface Options → I2C → Yes / Enable`  

**Only for connect the rasberry pi remotely:**  
    1. `Interface Options → SSH → Yes / Enable`  
    2. `Interface Options → RPI Connect|VNC → Yes / Enable`  


`/home/$USER/Desktop/yolo_hailo/yolov11l.hef`  
`Write each object class[bear,boar,deer,wolf,cow,person,dog] on a new line in your that file:'/home/$USER/Desktop/yolo_hailo/labels.txt'`  

## 📦 Setup  
1. `cd /home/$USER/Desktop/yolo_hailo`
2. `sudo apt update`
3. `sudo apt install -y python3-pip python3-libcamera python3-kms++ python3-prctl libatlas-base-dev`
4. `git clone -b next https://github.com/raspberrypi/picamera2.git`
5. `cd picamera2`
6. `pip install -e . --break-system-packages`
7. `cd examples/hailo`

## 🎉 Run
`python detect.py --model /home/$USER/Desktop/yolo_hailo/yolov11l.hef --labels /home/$USER/Desktop/yolo_hailo/labels.txt`
<details>
<summary>🛠️ Picamera2 and Camera Module 3 NoIR Compatibility</summary>
The following error is not caused by the HEF model. It occurs because the `detect.py` file from the GitHub repository is incompatible with the version of Picamera2 installed on the Raspberry Pi:

```text
ImportError: cannot import name 'hailo_architecture' from 'picamera2.devices'
```

The newer `detect.py` file uses the following import:

```python
from picamera2.devices import Hailo, hailo_architecture
```

However, `hailo_architecture` is not available in some Picamera2 versions installed through APT on Raspberry Pi OS.

In this situation, you have two options:

1. Use the compatible `detect.py` file included in this repository.
2. Modify your own `detect.py` file by following the steps below.

Because the model is explicitly provided using the `--model` argument, the `hailo_architecture` function is not required.

### 1. Verify that the base Hailo class works

```bash
python3 -c "from picamera2.devices import Hailo; print('Hailo import OK')"
```

You should receive the following output:

```text
Hailo import OK
```

### 2. Back up the `detect.py` file

First, navigate to the directory containing the Hailo examples:

```bash
cd "$HOME/Desktop/wildlife_YOLO/picamera2-main/examples/hailo"
```

Create a backup of the file:

```bash
cp detect.py detect_original.py
```

### 3. Fix the incompatible import

The following command removes the `hailo_architecture` import:

```bash
sed -i 's/from picamera2.devices import Hailo, hailo_architecture/from picamera2.devices import Hailo/' detect.py
```

Verify the change:

```bash
grep -n "picamera2.devices" detect.py
```

The result should look like this:

```python
from picamera2.devices import Hailo
```

### 4. Prevent the program from running without a model

Open the file:

```bash
nano detect.py
```

Find the following block:

```python
if args.model is None:
    if hailo_architecture() == 'HAILO10H':
        args.model = '/usr/share/hailo-models/yolov8m_h10.hef'
    else:
        args.model = '/usr/share/hailo-models/yolov8s_h8l.hef'
```

Replace the entire block with:

```python
if args.model is None:
    parser.error("The --model argument is required.")
```

Save and close the file:

```text
Ctrl+O
Enter
Ctrl+X
```

Verify the change:

```bash
grep -n -A 3 "if args.model is None" detect.py
```

You should see:

```python
if args.model is None:
    parser.error("The --model argument is required.")
```

### 5. Verify the Hailo device

```bash
hailortcli fw-control identify
```

If you are using a Hailo-8 or the 26 TOPS Raspberry Pi AI HAT+, the output should contain information similar to:

```text
Board Name: Hailo-8
Device Architecture: HAILO8
```

### 6. Verify the HEF file

```bash
hailortcli parse-hef "$HOME/Desktop/wildlife_YOLO/yolov11l.hef"
```

If this command cannot read the HEF file, check the compatibility between the HEF file and the version of HailoRT installed on the Raspberry Pi before running `detect.py`.

### 7. Check the label file

```bash
cat -n "$HOME/Desktop/wildlife_YOLO/labels.txt"
```

The classes must appear in the same class ID order that was used during model training:

```text
1  bear
2  boar
3  deer
4  wolf
5  cow
6  person
7  dog
```

If necessary, recreate the file in the correct order:

```bash
printf '%s\n' bear boar deer wolf cow person dog > "$HOME/Desktop/wildlife_YOLO/labels.txt"
```

The numbers shown above are only line numbers added by the `cat -n` command. Do not write these numbers into the `labels.txt` file.

### 8. Run the model

```bash
cd "$HOME/Desktop/wildlife_YOLO/picamera2-main/examples/hailo"

python3 detect.py \
    --model "$HOME/Desktop/wildlife_YOLO/yolov11l.hef" \
    --labels "$HOME/Desktop/wildlife_YOLO/labels.txt" \
    --score_thresh 0.45
```

## 📷 Camera Module 3 NoIR Blurry Image Fix

Camera Module 3 NoIR does not require a separate camera driver. If the camera is detected by the system but the image appears blurry, the most likely cause is that autofocus is not enabled in `detect.py`.

First, verify that the camera is detected by the system:

```bash
rpicam-hello --list-cameras
```

For Camera Module 3, the output should include `imx708`.

### 1. Import autofocus controls

Open the `detect.py` file:

```bash
nano detect.py
```

Add the following line to the import section at the beginning of the file:

```python
from libcamera import controls
```

An example import section should look like this:

```python
import argparse
import cv2

from libcamera import controls
from picamera2 import MappedArray, Picamera2, Preview
from picamera2.devices import Hailo
```

### 2. Modify the camera configuration

Find the following section:

```python
controls = {'FrameRate': 30}
config = picam2.create_preview_configuration(main, lores=lores, controls=controls)
picam2.configure(config)
```

Replace it with:

```python
camera_controls = {
    'FrameRate': 30
}

config = picam2.create_preview_configuration(
    main,
    lores=lores,
    controls=camera_controls
)

picam2.configure(config)

picam2.set_controls({
    "AfMode": controls.AfModeEnum.Continuous
})
```

The variable is deliberately named `camera_controls`. Using `controls` as the variable name would conflict with `libcamera.controls`.

All indentation in this section must use four spaces instead of Tab characters. Mixing tabs and spaces may produce errors such as:

```text
TabError: inconsistent use of tabs and spaces in indentation
```

```text
IndentationError: unexpected indent
```

Check the file for indentation and syntax errors:

```bash
python3 -m tabnanny detect.py
python3 -m py_compile detect.py
```

If both commands finish without producing any output, no indentation or Python syntax errors were detected.

### 3. Test the camera without the model

Before running the Hailo model, test continuous autofocus directly through the Raspberry Pi camera application:

```bash
rpicam-hello --timeout 0 --autofocus-mode continuous
```

If the image becomes sharp during this test but remains blurry in `detect.py`, the problem is related to the camera configuration in `detect.py`, not the camera hardware.

If the image remains blurry in both applications, check the following:

* Remove any protective film covering the camera lens.
* Clean the lens using a clean, soft microfiber cloth.
* Test autofocus using a real object instead of a computer or phone screen.
* Place the test object at least 30–50 cm away from the camera.
* Wait a few seconds after starting the camera to allow autofocus to settle.
* Point the camera toward a detailed, high-contrast object.

### About NoIR colors

Camera Module 3 NoIR does not contain an infrared-cut filter. Therefore, the following color differences are normal during daytime use:

* Purple or pink color casts
* Faded or altered green tones
* Unusual white balance
* Visible infrared light sources

These color differences are unrelated to image blur. If natural daytime colors are required, use a standard Camera Module 3 or an external IR-cut filter.

For nighttime operation, Camera Module 3 NoIR can be used with an 850 nm infrared illuminator.

## 🔄 Updating the Picamera2 Package

As an alternative to manually modifying `detect.py`, you can update Raspberry Pi OS and reinstall the Picamera2 package:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install --reinstall python3-picamera2
sudo reboot
```

Check the installed Picamera2 version:

```bash
python3 -c "import importlib.metadata; print(importlib.metadata.version('picamera2'))"
```

However, if the model is explicitly supplied using the `--model` argument, removing the unnecessary `hailo_architecture` dependency is sufficient for the current test.
</details>
