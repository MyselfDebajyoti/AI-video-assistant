# AI Video Assistant

AI Video Assistant turns a YouTube video or a local audio/video file into a searchable meeting brief. It downloads or converts the source, transcribes it, generates an AI summary and meeting title, extracts follow-up information, and lets you ask questions about the transcript through a retrieval-augmented chat experience.

The project is designed for meetings, interviews, lectures, demos, and other spoken-content recordings where reading or reviewing the entire video is time-consuming.

## What the application does

Given a YouTube URL or a local media-file path, the application:

1. Acquires the audio:
   - Downloads the best available audio from a URL with `yt-dlp`.
   - Converts a local audio/video file to mono, 16 kHz WAV with `pydub` and FFmpeg.
2. Splits the normalized WAV into 10-minute chunks.
3. Transcribes each chunk:
   - **English:** local OpenAI Whisper.
   - **Hinglish:** Sarvam AI speech-to-text translation, returning English text.
4. Uses Mistral to generate:
   - A short professional title.
   - A bullet-point meeting summary.
   - Action items, including owner and deadline when mentioned.
   - Key decisions.
   - Unresolved questions and follow-up topics.
5. Builds a local Chroma vector store from the transcript using Hugging Face embeddings.
6. Enables transcript-grounded question answering through a LangChain RAG chain.

The Streamlit UI presents the results as a dashboard with pipeline status, summary, transcript, extracted insights, and a chat history.

## Features

- YouTube URL input
- Local audio and video file input
- Local English transcription with configurable Whisper model size
- Hinglish-to-English transcription through Sarvam AI
- Automatic audio normalization and chunking for long recordings
- AI-generated meeting title
- Map/reduce-style transcript summarization for long recordings
- Action-item extraction with owner and deadline fields
- Key-decision extraction
- Open-question extraction
- Transcript-grounded RAG chat
- Local persistent Chroma vector database
- Streamlit dark-themed interface with live pipeline status
- Interactive CLI alternative
- Chat history and clear-chat controls during a Streamlit session

## Tech stack

| Area | Technology | Purpose |
| --- | --- | --- |
| UI | Streamlit | Interactive web application |
| Language | Python 3.10+ recommended | Application runtime |
| Video/audio acquisition | `yt-dlp` | Download YouTube audio |
| Audio processing | `pydub`, FFmpeg | Convert, normalize, and chunk audio |
| English speech-to-text | OpenAI Whisper, PyTorch | Local transcription |
| Hinglish speech-to-text | Sarvam AI API | Speech-to-text translation |
| LLM provider | Mistral AI | Titles, summaries, extraction, and answers |
| LLM orchestration | LangChain LCEL | Prompt and chain composition |
| Embeddings | `sentence-transformers` / `all-MiniLM-L6-v2` | Transcript embeddings |
| Vector database | Chroma | Similarity search for RAG |
| Configuration | `python-dotenv` | Load secrets from `.env` |

## Project structure

```text
.
├── app.py                    # Streamlit application
├── main.py                   # Interactive CLI pipeline and chat
├── test.py                   # Manual end-to-end example script
├── Requirements.txt          # Python dependencies
├── core/
│   ├── extractor.py          # Action items, decisions, and questions
│   ├── rag_engine.py         # Retrieval-augmented question answering
│   ├── summarizer.py         # Title and summary generation
│   ├── transcriber.py        # Whisper and Sarvam transcription routing
│   └── vector_store.py       # Chroma store and embedding setup
└── utils/
    └── audio_processor.py    # Downloading, conversion, and chunking
```

Runtime-generated files are written outside the source modules:

```text
downloades/                  # Downloaded audio and generated audio chunks
vector_db/                   # Persistent Chroma database
```

The `downloades` directory name is intentionally shown as it exists in the current implementation.

## Requirements

- Python 3.10 or newer is recommended.
- FFmpeg must be installed as a system executable. The Python `ffmpeg-python` package is only a wrapper and does not install the FFmpeg binary.
- A Mistral API key is required for title generation, summarization, extraction, and RAG answers.
- A Sarvam API key is required only when the `hinglish` transcription option is selected.
- The first English run downloads the configured Whisper model.
- The first RAG run downloads the `all-MiniLM-L6-v2` embedding model from Hugging Face.

### Installing FFmpeg

macOS with Homebrew:

```bash
brew install ffmpeg
```

Ubuntu/Debian:

```bash
sudo apt update
sudo apt install ffmpeg
```

Windows users can install FFmpeg and ensure its `bin` directory is available on `PATH`.

Verify the installation:

```bash
ffmpeg -version
```

## Installation

Clone or download the project, then create and activate a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate        # macOS/Linux
# .venv\Scripts\activate         # Windows PowerShell
```

Install the project dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install -r Requirements.txt
```

The source imports two LangChain integration packages directly. If they are not installed by the resolver on your platform, install them explicitly:

```bash
python -m pip install langchain-text-splitters langchain-chroma
```

## Configuration

Create a `.env` file in the project root:

```dotenv
MISTRAL_API_KEY=your_mistral_api_key
SARVAM_API_KEY=your_sarvam_api_key

# Optional Whisper configuration
WHISPER_MODEL=small

# Optional Sarvam configuration
SARVAM_STT_MODEL=saaras:v2.5
```

`MISTRAL_API_KEY` is used by all functions in `core/summarizer.py`, `core/extractor.py`, and `core/rag_engine.py`.

`SARVAM_API_KEY` is read when Hinglish mode is used. Sarvam audio is sent in pieces of up to 25 seconds because the synchronous endpoint has a 30-second limit.

Available Whisper model names generally include `tiny`, `base`, `small`, `medium`, and `large`. Larger models may improve transcription quality but require more memory and take longer to load.

Never commit `.env` or API keys to source control.

## Running the Streamlit application

From the project root, with the virtual environment active:

```bash
streamlit run app.py
```

Then open the local URL shown by Streamlit, usually `http://localhost:8501`.

### Using the UI

1. Enter a YouTube URL or a local file path in the sidebar.
2. Select `english` or `hinglish`.
3. Click **Analyse**.
4. Wait for the pipeline status indicators to complete.
5. Review the generated title, summary, transcript, action items, decisions, and open questions.
6. Ask questions in **Chat with your Meeting**.

Examples of supported inputs:

```text
https://www.youtube.com/watch?v=VIDEO_ID
/Users/you/recordings/team-sync.mp4
/Users/you/recordings/interview.wav
```

The local-file branch accepts formats supported by `pydub`/FFmpeg. The file must exist and be readable by the process.

## Running the CLI

The CLI runs the same core workflow and starts an interactive transcript chat:

```bash
python main.py
```

When prompted:

```text
Enter YouTube URL or local file path:
Language (english/hinglish):
```

After processing, the CLI prints the title, summary, action items, key decisions, and open questions. It then accepts questions until `exit`, `quit`, or `q` is entered.

## Pipeline architecture

```text
YouTube URL / local file
          │
          ▼
download_youtube_audio() or convert_to_wav()
          │
          ▼
chunk_audio()  ── 10-minute WAV chunks
          │
          ▼
transcribe_all()
   ├── english  ── local Whisper
   └── hinglish ── Sarvam API → English transcript
          │
          ├── generate_title()       ── Mistral
          ├── summarize()            ── chunk summaries + final summary
          ├── extract_action_items()
          ├── extract_key_decisions()
          ├── extract_questions()    ── Mistral
          │
          └── build_rag_chain()
                    │
                    ├── split transcript into 500-character chunks
                    ├── embed with all-MiniLM-L6-v2
                    ├── persist in Chroma
                    └── retrieve top 4 chunks for each question
```

### Audio processing

`process_input()` chooses the acquisition path based on whether the source starts with `http://` or `https://`. Local files are converted to a mono 16 kHz WAV file, then every input is divided into 10-minute chunks. Chunk files are retained in the download directory after processing.

### Transcription

Whisper is loaded lazily and cached in memory for the current process. Hinglish mode uses Sarvam's `speech-to-text-translate` endpoint and joins the translated output from 25-second WAV pieces.

### Summarization and extraction

Long transcripts are split into 3,000-character sections with 200-character overlap. Each section is summarized by Mistral, and the partial summaries are combined into the final bullet-point summary. Extraction prompts separately identify action items, decisions, and unresolved questions.

### Retrieval-augmented chat

The transcript is split into 500-character sections with 50-character overlap, embedded locally, and stored in Chroma under the `meeting_transcript` collection. Each question retrieves the four most similar chunks. The Mistral prompt instructs the assistant to answer only from retrieved transcript context and to explicitly say when the information cannot be found.

## Data and persistence

- `downloades/` contains downloaded audio, converted files, and chunk WAV files.
- `vector_db/` contains the persistent Chroma store.
- Streamlit session state contains the current result and chat history for the active browser session.
- No database migration or user-account system is included.
- The current pipeline does not automatically clean up source audio or chunk files.

To start a fresh RAG collection, stop the application and remove only the project-generated `vector_db/` directory, then run a new analysis. Do not remove files you need to keep.

## Testing and development

`test.py` is currently a manual end-to-end script with a hard-coded YouTube URL. It exercises acquisition, transcription, title generation, summarization, and extraction, but it is not an automated unit-test suite and has no assertions.

Run it only after configuring the required API keys and replacing the example URL if needed:

```bash
python test.py
```

For a quick import/syntax check:

```bash
python -m compileall app.py main.py core utils
```

## Troubleshooting

### `ffmpeg` or decoder errors

Install the FFmpeg system binary and verify that `ffmpeg -version` works in the same terminal where Streamlit or Python is started.

### Mistral authentication errors

Confirm that `.env` is in the project root and contains a valid `MISTRAL_API_KEY`. The application calls `load_dotenv()` at startup, so restart the process after changing the file.

### Hinglish transcription fails

Set `SARVAM_API_KEY` and check that the key can access Sarvam's speech-to-text translation endpoint. English mode does not require this key.

### Whisper is slow or runs out of memory

Set a smaller model before starting the process:

```dotenv
WHISPER_MODEL=base
```

The default is `small`.

### Model downloads or RAG startup are slow

Whisper and the Hugging Face embedding model are downloaded on first use. Later runs can reuse the local model caches, depending on the environment.

### `ModuleNotFoundError` for LangChain integrations

Install the direct imports explicitly:

```bash
python -m pip install langchain-text-splitters langchain-chroma
```

## Current limitations

- Processing is synchronous; long videos can take significant time.
- There is no speaker diarization in the implemented pipeline.
- The interface accepts a file path rather than a Streamlit upload widget.
- The RAG collection uses a fixed collection name and directory, so multiple analyses are not isolated automatically.
- The CLI and UI do not export summaries to PDF or TXT, even though PDF-related libraries are present in `Requirements.txt`.
- YouTube availability, region restrictions, authentication, and provider quotas can affect processing.
- The generated HTML result cards render model output directly; treat generated content as untrusted when extending the UI.

## License

No license file is currently included. Add a license before distributing the project or accepting external contributions.
