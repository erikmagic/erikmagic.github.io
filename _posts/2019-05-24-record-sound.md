---
layout: post
title: Record your computer Audio using Python 
date: 2019-05-26 19:22
summary: Let's put the pieces together and automatically record one's computer audio out.
categories: software
---

One can record one's audio out using software like audacity. Unfortunately, these software are limited and cannot be integrated into larger software pipelines to record high quality and complex tasks. This post will show you how to use various software on windows to records one's audio output and save it in high quality .wav file.  
Let's get started with the modules and software we will need.  

* Install [VB-Cable Virtual Audio Device](https://www.vb-audio.com/Cable/), Then reboot. This software creates a pipe from audio-out to audio-in. So when youtube or whatever plays, the output from the soundcard is multiplexed to an input channel (like a fake microphone). We can record on this input channel later on. 
* Create a new Conda environment, and activate it.
* Pip install the packages wave, pydub, requests and tqdm. Wave is used to save the audio as wav file. Pydub is used to manipulate the audio signals (like cutting leading silences).  Finally, TQDM is used only to check the progress of the recording.
* Also install the PyAudio wheel. For windows, you can find the wheel [here](https://www.lfd.uci.edu/~gohlke/pythonlibs/#pyaudio). PyAudio is used to create the stream to record the artificial input channel. Copy the wheel to some kind of `resource` folder in your directory, and then pip install like a normal package from pypi.
* Modify the line 868 in "C:\Users\<you>\.conda\envs\rsc\Lib\site-packages\pydub\audio-segment.py"
for instance, adding shell=True to the arguments. This solves a bug for pydub on windows... Notice this location is the one used for the pydub pacakage in your conda environment.
* Download the FFmpeg framework. The ffmpeg framework is the leading multimedia framework to decode, encode, transcode, mux, demux, stream, filter and play. We will use it with pyaudio/pydub.
* unzip the ffmpeg framework files and save them to your `resources` folder in your directory.
* add the /resources/ffmpeg.../bin to environment variables and reboot.
* Also add the libav audio processing tools to your `resource` folder. You can download the [libav](https://libav.org/download/) library.  
  
Okay, to summarize, at this point you dowloaded vbcable, the pyaudio wheel, the ffmpeg framework and the libav audio processing tools from the internet. You also created a conda environment and installend the other python packages needed from pypi. Let's get started with the code!  
We first need to import a couple of modules:  
```python
import requests
import sys
import json
import os

# add the necessary packages (ffmpeg.exe and libav so python (more importantly
# some of the imported packages) can use it.
os.environ['PATH'] += os.path.join(config['root_path'],
			 'resources', 'libav', 'bin')
dir_path = os.path.dirname(os.path.realpath(__file__))

os.environ['PATH'] += os.path.join(dir_path,
				 'resources',
				 'ffmpeg-4.1.3-win64-static',
				 'bin',
				 'ffmpeg.exe')
import wave
from tqdm import tqdm
from pydub import AudioSegment
import pyaudio

# instantiate the few constants that we will use
SPLIT = 10

# the output path will be where the audio will eventually be saved at
output_path = os.path.join('C:\\', 'users', os.getlogin(), 'Music', 'playlists')
if not os.path.exists(output_path):
    os.mkdir(output_path)
```
Now that the necessary modules have been imported and that we've instantiated the necessary variables, let's start by writing the helper functions. 































