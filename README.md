## 👀 run-cv-model-in-rp5 Overview         3/3  
<h1 align="center">Run computer vision model in Rasberry Pi 5 AI HAT</h1>  


## 🔎 Preparation
`/home/$USER/Desktop/yolo_hailo/yolov11n.hef`  
`Write each object class[rock,paper,scissors] on a new line in your that file:'/home/$USER/Desktop/yolo_hailo/labels.txt'`  

## 📦 Setup  
1. `cd /home/$USER/Desktop/yolo_hailo`
2. `sudo apt update`
3. `sudo apt install -y python3-pip python3-libcamera python3-kms++ python3-prctl libatlas-base-dev`
4. `git clone -b next https://github.com/raspberrypi/picamera2.git`
5. `cd picamera2`
6. `pip install -e . --break-system-packages`
7. `cd picamera2/examples/hailo`
8. `python detect.py --model /home/$USER/Desktop/yolo_hailo/yolov11n.hef --labels /home/$USER/Desktop/yolo_hailo/labels.txt`

