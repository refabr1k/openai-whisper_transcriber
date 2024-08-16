# openai-whisper_transcriber
Simple openai-whisper transcriber to translate audio recordings (focused for simplified chinese) script served on a python script

## Readme:

1. setup whisper (see https://gist.github.com/refabr1k#setting-up-torch-with-cuda-if-you-have-a-nvidia-gfx-card)
2. Save the 2 files below in the same folder and run `python server.py`

## server.py
```
from flask import Flask, request, jsonify, send_file, send_from_directory
import whisper
import os

app = Flask(__name__)

# Load the Whisper model
model = whisper.load_model("large-v3")

def format_time(seconds):
    """Convert seconds to MM:SS format."""
    minutes, seconds = divmod(seconds, 60)
    return f"{int(minutes):02}:{int(seconds):02}"

@app.route('/')
def index():
    return send_from_directory('.', 'index.html')

@app.route('/transcribe', methods=['POST'])
def transcribe():
    file = request.files.get('file')
    if not file:
        return jsonify({'error': 'No file uploaded'}), 400

    language = request.form.get('language', 'zh')
    translate = request.form.get('translate', 'false').lower() == 'true'
    
    input_file_path = 'temp_audio.wav'
    file.save(input_file_path)

    if language == 'zh':
        prompt = '以下是普通话的句子'
        result = model.transcribe(
            whisper.load_audio(input_file_path), 
            language=language, 
            verbose=True, 
            initial_prompt=prompt,
            task="translate" if translate else "transcribe"
        )
    else:
        result = model.transcribe(
            whisper.load_audio(input_file_path), 
            language=language, 
            verbose=True, 
            task="translate" if translate else "transcribe"
        )

    output_file_path = 'transcription.txt'
    with open(output_file_path, 'w', encoding='utf-8') as output_file:
        for segment in result['segments']:
            start_time = segment['start']
            end_time = segment['end']
            text = segment['text']
            transcribed_line_text = f"[{format_time(start_time)} - {format_time(end_time)}] {text}\n"
            output_file.write(transcribed_line_text)

    os.remove(input_file_path)

    return send_file(output_file_path, as_attachment=True)

if __name__ == '__main__':
    app.run()

```


## index.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Buu's Audio Transcriber</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background-color: #f0f0f0;
        }
        .container {
            width: 80%;
            max-width: 600px;
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }
        input[type="file"], input[type="text"], input[type="checkbox"] {
            margin-bottom: 10px;
        }
        #progress {
            margin-top: 10px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Buu's Audio Transcriber</h1>
        <form id="transcription-form">
            <label for="file">Select Audio File:</label>
            <input type="file" id="file" name="file" accept="audio/*" required><br>
            <label for="language">Language (use 'zh' for chinese. 'en' for english):</label>
            <input type="text" id="language" name="language" placeholder="zh" value="zh"><br>
            <label for="translate">Translate to English:</label>
            <input type="checkbox" id="translate" name="translate"><br>
            <button type="submit">Submit</button>
        </form>
        <div id="progress"></div>
        <a id="download-link" href="#" style="display: none;">Download Transcription</a>
    </div>

    <script>
        document.getElementById('transcription-form').addEventListener('submit', async (event) => {
            event.preventDefault();

            const formData = new FormData();
            formData.append('file', document.getElementById('file').files[0]);
            formData.append('language', document.getElementById('language').value);
            formData.append('translate', document.getElementById('translate').checked);

            const progress = document.getElementById('progress');
            const downloadLink = document.getElementById('download-link');

            progress.textContent = 'Uploading...';

            try {
                const response = await fetch('/transcribe', {
                    method: 'POST',
                    body: formData
                });

                if (response.ok) {
                    const blob = await response.blob();
                    const url = URL.createObjectURL(blob);
                    downloadLink.href = url;
                    downloadLink.download = 'transcription.txt';
                    downloadLink.style.display = 'block';
                    downloadLink.textContent = 'Download Transcription';
                    progress.textContent = 'Transcription complete!';
                } else {
                    progress.textContent = 'An error occurred during transcription.';
                }
            } catch (error) {
                progress.textContent = 'An error occurred: ' + error.message;
            }
        });
    </script>
</body>
</html>
```

![image](https://github.com/user-attachments/assets/56e534b9-ebf1-4527-ae6a-0d180ffeac22)

Transcribed file will be available to download after transcription.
