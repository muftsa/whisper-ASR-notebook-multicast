# whisper-ASR-notebook-multicast

To pull in a multicast audio feed and transcribe it with Whisper in this Jupyter notebook, we’ll need to capture the audio stream, save it to a file, and then process it using Whisper. Here’s how to set it up.

### Requirements:
1. **Install additional libraries**:
   - `pyaudio`: For capturing audio streams.
   - `ffmpeg-python`: For handling audio stream processing.
   - `openai-whisper`: For transcription.

   ```bash
   pip install pyaudio ffmpeg-python openai-whisper
   ```

2. **Network Configuration**:
   Ensure the multicast source IP, port, and network permissions are configured to allow receiving the multicast stream.

### Code

This notebook will:
1. Capture a multicast audio stream and save it temporarily.
2. Use Whisper to transcribe the saved audio.

```python
# Import necessary libraries
import socket
import wave
import pyaudio
import ffmpeg
import whisper
import tempfile
import os

# Setup multicast parameters
MULTICAST_IP = '239.255.0.1'  # Replace with your multicast IP address
PORT = 5004                    # Replace with your multicast port
BUFFER_SIZE = 4096

# Whisper model
model = whisper.load_model("base")

# Function to receive multicast audio and save to a file
def capture_multicast_audio(duration=10, output_path="multicast_audio.wav"):
    """
    Captures audio from a multicast IP and saves it as a WAV file.
    :param duration: Duration to capture in seconds.
    :param output_path: Path to save the audio file.
    """
    # Set up multicast socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('', PORT))
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, 
                    socket.inet_aton(MULTICAST_IP) + socket.inet_aton('0.0.0.0'))

    # Initialize PyAudio for WAV file saving
    audio = pyaudio.PyAudio()
    stream = audio.open(format=pyaudio.paInt16, channels=1, rate=44100, output=True)

    frames = []
    print(f"Capturing audio from multicast stream at {MULTICAST_IP}:{PORT} for {duration} seconds...")

    try:
        for _ in range(0, int(44100 / BUFFER_SIZE * duration)):
            data = sock.recv(BUFFER_SIZE)
            frames.append(data)
            stream.write(data)
    finally:
        stream.stop_stream()
        stream.close()
        audio.terminate()
        sock.close()

    # Save the captured audio to a file
    with wave.open(output_path, 'wb') as wf:
        wf.setnchannels(1)
        wf.setsampwidth(audio.get_sample_size(pyaudio.paInt16))
        wf.setframerate(44100)
        wf.writeframes(b''.join(frames))
    print(f"Audio saved to {output_path}")

# Capture audio and transcribe
temp_audio_file = tempfile.NamedTemporaryFile(suffix=".wav", delete=False).name

# Capture audio
capture_multicast_audio(duration=10, output_path=temp_audio_file)

# Transcribe the captured audio
print("Transcribing audio...")
result = model.transcribe(temp_audio_file)

# Display transcription
print("Transcription:")
print(result["text"])

# Clean up the temporary file
os.remove(temp_audio_file)
```

### Explanation

1. **Capture Audio from Multicast**: 
   - Creates a UDP socket to receive packets from the specified multicast IP and port.
   - Collects audio data for the specified `duration`.
   - Saves this data to a temporary `.wav` file.

2. **Transcribe with Whisper**: 
   - Loads the saved audio and uses the Whisper model to transcribe the content.
   
3. **Display Transcription**: 
   - Prints the transcribed text.

This setup allows you to capture and transcribe audio directly from a multicast stream in your Jupyter notebook! Make sure the notebook has the appropriate permissions and network configuration for receiving multicast packets.
