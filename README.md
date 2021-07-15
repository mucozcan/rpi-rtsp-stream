# How to setup an RTSP server on Headless Raspberry Pi and stream RPi Camera data?


## 1. Setting up a new Raspberry Pi

This guide is tested on 'Raspbian', so my suggestion is you should download the latest Raspbian version and flash it to SD card(Min. 16 GB suggested.).

Raspbian download page : https://www.raspberrypi.org/software/operating-systems/ \
You can use 'Raspberry Pi Imager' for flashing Raspbian to your SD Card. You can find requred installation file for your OS in [software page](https://www.raspberrypi.org/software/). Another good option is [Etcher](https://etcher.download/)


After successfully flashing Raspbian to your SD Card, don't remove it from your PC. Since our RPi setup will be headless, we need to enable SSH before powering the RPi.

* First, you'll have to locate the boot directory. If you're on Ubuntu, it's in `/media/your-hostname/boot`. Open terminal and type:  
    ```
    $ cd /media/your-hostname/boot
    ```

* After navigating boot directory,  all you have to do is  creating an empty file called ssh.

    ```
    $ touch ssh
    ```

* Insert SD card to the Raspberry Pi. Now connect the RPi to power source and to your PC via ethernet cable. 

## 2. Connecting to RPi via SSH

* After connecting the RPi and your PC via ethernet, we should enable the hotspot on our pc. Open a terminal and type:

    ```
    $ nm-connection-editor
    ```
    
    Double click to existing `Ethernet` connection to edit configuration. In opened window, navigate to `IPv4 Settings`, click `Method` options and select `Shared to other computers`. 

* type `ifconfig` and check your ethernet interface name. 

* To scan your ethernet interface, you can use `arp-scan`. 

    ```
    $ sudo apt-get install -y arp-scan
    ```

* Scan your ethernet interface to find RPi's IP address. Only one result should be returned.

    ```
    $ sudo arp-scan --interface=your-ethernet-interface --localnet
    ```

* Connect to RPi. default username is `pi` and default password is `raspberry`

    ```
    $ ssh pi@[RPi IP Address]
    ```

## 3. Installing necessary packages to the RPi

* After successfully connecting to the RPi via SSH, execute the following instructions on the command line to download and install the latest kernel, GPU firmware, and applications:

    ```
    $ sudo apt-get update
    ```
    
    ``` 
    $ sudo apt full-upgrade
    ```

* Now you need to enable camera support using the raspi-config program you will have used when you first set up your Raspberry Pi.

    ```
    $ sudo raspi-config
    ```
    
    Use the cursor keys to select and open Interfacing Options, and then select Camera and follow the prompt to enable the camera.

    Upon exiting `raspi-config`, it will ask to reboot. The enable option will ensure that on reboot the correct GPU firmware will be running with the camera driver and tuning, and the GPU memory split is sufficient to allow the camera to acquire enough memory to run correctly.

* After rebooting, connect to the RPİ via SSH again.

* Install development tools: 

    ```
    $ sudo apt-get install git build-essential autoconf automake autopoint libtool pkg-config -y

    $ sudo apt-get install gtk-doc-tools libglib2.0-dev -y

    $ sudo apt-get install checkinstall
    
    ```

* To install essential GStreamer tools to the RPi, run the following command:

    ```
    $ sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-doc gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-pulseaudio 
    
    ```

* Now we need to install RTSP support of GStreamer:
    
    ```
    $ sudo apt-get install libgstrtspserver-1.0-dev gstreamer1.0-rtsp
    ```
    
    Clone the repository:

    ```
    $ git clone https://github.com/GStreamer/gst-rtsp-server.git
    $ cd gst-rtsp-server/
    ```

    we need to checkout previous version and then compile:
    
    ```
    $ git checkout 1.13.91
    $ ./autogen.sh
    $ ./configure
    $ make
    $ sudo checkinstall make install # enter 3 and fill *Version* field with 1.13.91
    
    ```
* Now we need `raspivid` wrapper `gst-rpicamsrc`

    ```
    $ cd ~
    $ sudo apt-get install git
    $ git clone https://github.com/thaytan/gst-rpicamsrc.git
    $ cd gst-rpicamsrc/
    $ ./autogen.sh 
    $ make
    $ sudo make install
    ```

    Check,  if `gst-rpicamsrc` is installed:
    ```
    $ gst-inspect-1.0 | grep rpicamsrc
    ```

## 4. Stream camera data over RTSP Server

* For the sake of simplicty, we are gonna use GStreamer examples that we've already installed, but you can write your own RTSP Server by following documentation. Navigate to GStreamer RTSP examples:
    ```
    $ cd ~/gst-rtsp-server/examples/
    $ ./test-launch --gst-debug=3 "( rpicamsrc bitrate=8000000 awb-mode=tungsten preview=false ! video/x-h264, width=640, height=480, framerate=30/1 ! h264parse ! rtph264pay name=pay0 pt=96 )" 
    ```
    If you have an advanced knowledge about media codecs and GStreamer elements, feel free to edit the launchline above.

    After executing you should see the following output: `stream ready at rtsp://127.0.0.1:8554/test`


## 5. Connecting to the server and retrieve data

* Now our RTSP server should be up and running. There are a lot of ways to connect to server and retrieve data to your local machine. First, we are gonna use VLC media utilites and then we capture the stream by using OpenCV.

* Install VLC Media Player if you haven't already. On your local machine:

    ```
    $ sudo apt-get install VLC
    ```

* Open up VLC and right click on window: Click `Open Media` -> `Open Network`. Enter the RTSP url : `rtsp://[Your RPi's IP Address]:8554/test`. (You can check your RPi's ip address by typing `ifconfig` in RPi terminal. You need to look IPv4 address of ethernet interface). Click on `Play` and you should be able to getvideo stream from your RPi cam!

* To retrieve video stream using OpenCV, install OpenCV Python bindings, if you haven't already:
    ```
    $ pip3 install opencv-python
    ```

* Execute the following script to retrieve stream:

    ```python
    import cv2 

    cap = cv2.VideoCapture("rtsp://[Your RPi's IP Address]:8554/test")

    while True:
        _, frame  = cap.read()
                
        cv2.imshow("stream", frame)
        cv2.waitKey(1)          

    ```

