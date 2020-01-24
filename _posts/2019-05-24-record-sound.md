---
layout: post
title: Record your computer Audio using Python 
date: 2019-05-26 19:22
summary: Let's put the pieces together and automatically record one's computer audio out.
categories: software
---

One can record one's audio out using software like audacity. Unfortunately, these software are limited and cannot be integrated into larger software pipelines to record high quality and complex tasks. This post will show you how to use various software on windows to records one's audio output and save it in high quality .wav file.  
Let's get started with the modules and software we will need.  

* Install [VB-Cable Virtual Audio Device](https://www.vb-audio.com/Cable/), Then reboot. This software creates a pipe from audio-out to audio-in. So when youtube or whatever plays, the output from the soundcard is multiplexed to an input channel (like a fake microphone). We can record on this input channel later on. You can google how to use this software but it's pretty simple. Just change your window's playback device to CABLE Input(VB-Audio Virtual Cable).
* Create a new Conda environment, and activate it.
* Pip install the packages wave, pydub, requests and tqdm. Wave is used to save the audio as wav file. Pydub is used to manipulate the audio signals (like cutting leading silences).  Finally, TQDM is used only to check the progress of the recording.
* Also install the PyAudio wheel. For windows, you can find the wheel [here](https://www.lfd.uci.edu/~gohlke/pythonlibs/#pyaudio). PyAudio is used to create the stream to record the artificial input channel. Copy the wheel to some kind of `resource` folder in your directory, and then pip install like a normal package from pypi.
* Modify the line 868 in "C:\Users\<you>\.conda\envs\rsc\Lib\site-packages\pydub\audio-segment.py"
for instance, adding shell=True to the arguments. This solves a bug for pydub on windows... Notice this location is the one used for the pydub pacakage in your conda environment.
* Download the FFmpeg framework. The ffmpeg framework is the leading multimedia framework to decode, encode, transcode, mux, demux, stream, filter and play. We will use it with pyaudio/pydub.
* unzip the ffmpeg framework files and save them to your `resources` folder in your directory.
* add the /resources/ffmpeg.../bin to environment variables and reboot.
  
Okay, to summarize, at this point you dowloaded vbcable, the pyaudio wheel, the ffmpeg framework and the libav audio processing tools from the internet. You also created a conda environment and installend the other python packages needed from pypi. Let's get started with the code!  
We first need to import a couple of modules:  
```python
import requests
import sys
import json
import os

# add the necessary packages (ffmpeg.exe and libav so python 
#(more importantly some of the imported packages) can use it.
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

# the output path will be where the audio will eventually be 
# saved at
output_path = os.path.join('C:\\', 'users', os.getlogin(), 
				'Music', 'playlists')
if not os.path.exists(output_path):
    os.mkdir(output_path)
```
Now that the necessary modules have been imported and that we've instantiated the necessary variables, let's start by writing the helper functions. 
```python
def load_and_trim(filename: str) -> AudioSegment:
    """
    This functions loads multiple saved .wav files,
    concatenate them together and then applies a function
    that trims the silences at the beginning/end
    of the audio signal.
    Args:
        filename (str): root of the filename 
            (expects to be followed  by some
             number and the .wav extension)
    Returns:
        (AudioSegment): concatenate audio files with
            leading/ending silences stripped.
    """
    sound =  AudioSegment.from_file(filename + "0.wav",
                                    format="wav")
    for i in range(1, SPLIT):
        sound += AudioSegment.from_file(filename + str(i)
                    + ".wav", format="wav")

    start_trim = detect_leading_silence(sound[200:])
    end_trim = detect_leading_silence(sound.reverse())

    duration = len(sound)
    trimmed_sound = sound[start_trim:duration - end_trim]
    return trimmed_sound

def clean_temp_files(filename: str):
    """
    This function simply removes all the temporaryfiles which 
    were created by this script. These files obeys the pattern
    `filename` + {i}.wav.
    Args:
        filename (str): The filename which serves as the root
        name for all these temporary files.
    """
    for fl in glob.glob(filename + "*.wav"):
        os.remove(fl)


def detect_leading_silence(sound: AudioSegment, 
                           silence_threshold: float=-50.0,
                           chunk_size: int=21) -> int:
    '''
    This functions loads up an audio segment and detects 
    the length of the leading silence in ms. Iterates 
    over chunks until one with sound is detected
    Args:
        sound (AudioSegment): input pydub.AudioSegment sound
        silence_threshold (float): this is the threshold 
        decibel value to consider some sound as being `silent`
        chunk_size (int): size of an audio chunk which is 
        an argument to pyaudio/pydub , in ms.
    Returns:
        (int): The point the sound is no longer silent.
        In other words, the first chunk with a detected 
        sound above the threshold.
    '''
    trim_ms = 0  # ms

    assert chunk_size > 0  # to avoid infinite loop
    while sound[trim_ms:trim_ms + chunk_size].dBFS < silence_threshold
          and trim_ms < len(sound):
        trim_ms += chunk_size

    return trim_ms

```
Alright! Now, we have the helper methods necessary to actually record one's playback device. Let's code this function.
```python
def rip_single_sound(duration: int , filename_wav: str,
                     master_format: bool =False):
    """
    Record a sound from the playback device for the length
    specified. Does not return anything as it saves to 
    file the recording.
    Args:
        duration (int): The duration for which to record
            the sound.
        filename_wav (str): Root of the filename to store
            the sound. The .wav extension will be added
            automatically.
        master_format (bool): The master format flags
            change the quality of the recording to a
            sample rate of 96k hz.
    """
    # time_buffer adds some seconds to the recording
    # length to be able to save things up.
    time_buffer = 5
    # the chunk is like a buffer that stores samples
    chunk = 4096
    # Left and right channeles
    channels = 2
    # sample rate
    fs = 0
    if master_format:
        fs = 96000
        sample_format = pyaudio.paInt24
    else:
        fs = 44100
        sample_format = pyaudio.paInt16
    seconds = duration + time_buffer

    p = pyaudio.PyAudio()  # Create an interface 
                           # to PortAudio

    print('Recording')

    stream = p.open(format=sample_format,
                    channels=channels,
                    rate=fs,
                    frames_per_buffer=chunk,
                    input=True)

    frames = []  # Initialize array to store frames

    # Store data in chunks for the duration 
    # (plus buffer)
    total = int(fs / chunk * seconds)
    for i in tqdm(range(0, total)):
        data = stream.read(chunk)
        frames.append(data)

    # Stop and close the stream
    stream.stop_stream()
    stream.close()
    # Terminate the PortAudio interface
    p.terminate()

    print('Finished recording, now saving to file.')

    # Save the recorded data as a WAV file
    # (split in 10 files)
    individual_file_size = len(frames) // SPLIT
    current_position_to_write = 0
    for i in range(SPLIT):
        with wave.open(filename_wav + str(i)
                    + ".wav", 'wb') as wf:
            wf.setnchannels(channels)
            wf.setsampwidth(
                    p.get_sample_size(sample_format)
                )
            wf.setframerate(fs)
            if i < SPLIT - 1:
                wf.writeframes(b''.join(
                    frames[current_position_to_write:\
                           current_position_to_write
                           + individual_file_size]
                    )
                )
            else:
                wf.writeframes(
                    b''.join(
                        frames[
                            current_position_to_write:
                            ]
                        )
                    )
            current_position_to_write +=\
                    individual_file_size


    del frames

```
That's it! Now you can use the method `rip_single_sound` and you have a few seconds 
to record what you want to record on your computer. Then, just use the method
`load_and_trim` to clean your sound.
