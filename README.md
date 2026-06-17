# Audio Editor

A modern and lightweight Audio Editor that helps users quickly analyze, visualize, and manipulate audio files with an intuitive interface.

## Features

- Upload and process audio files directly in the browser
- Automatically extract audio metadata and file information
- View detailed audio properties including duration, bitrate, codec, and tags
- Edit and update MP3 metadata (ID3 tags)
- Shape and manipulate audio content using FFmpeg WebAssembly
- Reset modifications and restore the original state
- Fast client-side processing with no server dependency
- Modern and responsive Vue.js interface

### 🎵 Automatic Audio File Analysis

* Upload audio files and automatically extract key information.
* Detect file properties such as duration, format, sample rate, channels, and bitrate.
* Generate a quick overview without manual inspection.

### 📊 Audio Waveform Visualization

* Display audio waveforms in real time.
* Provide a visual representation of sound patterns.
* Enable easier audio inspection and navigation.

### ✂️ Audio Shaping & Editing

* Trim and shape audio segments.
* Modify selected portions of an audio track.
* Create cleaner and more focused audio outputs.

### 🔄 Reset Functionality

* Instantly restore the original audio state.
* Undo all applied modifications with a single click.
* Simplify experimentation without the risk of losing the source file.

### ⚡ User-Friendly Experience

* Clean and responsive interface.
* Fast audio processing.
* Suitable for both beginners and advanced users.

## Use Cases

* Audio inspection and analysis
* Quick waveform visualization
* Basic audio editing and shaping
* Educational and demonstration purposes
* Audio preprocessing workflows

## Technologies Used

- Vue 3
- Vite
- FFmpeg WebAssembly (@ffmpeg/ffmpeg)
- Music Metadata Parser (music-metadata)
- Browser ID3 Writer
- JavaScript (ES6+)
- HTML5 & CSS3

## Getting Started

1. Clone the repository.
2. Install dependencies.
3. Run the development server.
4. Upload an audio file and start editing.

## License

This project is licensed under the MIT License.
