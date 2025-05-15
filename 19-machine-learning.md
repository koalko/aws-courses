# Machine Learning 101

## Amazon Comprehend

Natural Language Processing (NLP) service. Uses document (text) as an input; outputs entities, phrases, language, PII, sentiments.

Based on either pre-trained or custom models. Able to do a realtime analysis for small workloads or async jobs for larger workloads. Service can be used from console or CLI. You can also use Comprehend API from an application.

## Amazon Kendra

Intelligent search service, designed to mimic interacting with a human expert. Supports wide range of question types:

- `Factoid`: Who, what, where
- `Descriptive`: How do I get my cat to stop being a jerk?
- `Keyword`: What time is the keynote address? (address can have multiple meanings; Kendra helps to determine intent)

Key concepts:

- `Index` - searchable data, organized in an efficient way
- `Data Source` - where your data lives; Kendra connects and indexes from this location; can be:
  - `S3`
  - `Confluence`
  - `Google Workspace`
  - `RDS`
  - `OneDrive`
  - `Salesforce`
  - `Kendra Web Crawler`
  - `Workdocs`
  - `FSX`
- Synchronize with index based on a `schedule`
- `Documents`: structured (FAQs), unstructured (HTML, PDF, text)
- Integrates with AWS Services: `IAM`, `Identity Center` (SSO), ...

## Amazon Lex

Provides text or voice conversational interfaces. Powers the Alexa service. Provides automatic speech recognition (ASR) and Natural Language Understanding (NLU).

Scales, integrates, quick to deploy, pay as you go pricing.

Use cases:

- Chatbots
- Voice Assistants
- Q&A Bots
- Info/Enterprise Bots

Concepts:

- Lex provides `bots`, conversing in 1+ languages
- `Intent` is an action the user wants to perform (_order a pizza_)
- `Utterances` – ways in which an intent might be said (_I want to order a pizza_)
- How to fulfil the intent (often using `Lambda` integration)
- `Slot` – parameters for an intent (_small/medium/large_)

## Amazon Polly

Converts text into "life-like" speech (in the same language, no translation). Two modes:

- Standard TTS: Concatenative (phonemes)
- Neural TTS: phonemes -> spectrograms -> vocoder -> audio (much more human/natural sounding, but more complex)

Output formats:

- MP3
- Ogg Vorbis
- PCM

Polly is capable of using Speech Synthesis Markup Language (SSML), which gives additional control over how Polly generates speech (emphasis, pronunciation, whispering, newscaster speaking style, etc.).

## Amazon Rekognition

Deep learning image and video analysis. Allow to identify objects, people, text, activities; can help with content moderation, face detection, face analysis, face comparison, pathing & much more.

Pay as you use: per image or per minute (video) pricing. Integrates with applications via API; event-driven.

Can analyze live video streams (for example, Kinesis video streams).

Example flow:

- image uploaded to `S3`
- `S3` event invokes `Lambda`
- `Lambda` uses `Rekognition` to identify animals
- metadata stored in `DDB`

## Amazon Textract

Used to detect and analyze text, contained in input documents (JPEG, PNG, PDF, TIFF). Outputs extracted text, structure and analysis. For the most documents the processing is synchronous (real-time). For large documents the processing is asynchronous.

Pay for usage; custom pricing available for large volume.

Use cases:

- detection of text
- relationship between text
- metadata (where text occurs)
- document analysis (names, address, birthdate)
- receipt analysis (prices, vendor, line items, dates)
- identity documents (abstract fields, for example `DocumentID`)

## Amazon Transcribe

Automatic Speech Recognition (ASR) service. Input: audio, output: text. Features:

- language customization
- filters for privacy
- audience-appropriate language
- speaker identification

Supports custom vocabularies and language models.

Pay as you use (per second of transcribed audio).

Use cases:

- full-text indexing of audio (allow searching)
- meeting notes
- subtitles/captions/transcripts
- call analytics (characteristics, summarization, categories and sentiment)
- integration with other apps / AWS ML services

## Amazon Translate

ML-based text translation service. Translates text from native language to other languages. Translation process:

1. Encoder reads source text and outputs semantic representation
2. Decoder reads semantic representation and writes target language

Attention mechanisms ensure "meaning" is translated.

Able to auto detect source text language.

Use cases:

- multilingual user experience for meeting notes, posts, communications, articles, emails, in-game chat, customer live chat
- translate incoming data (social media / news / communications)
- language-independence for other AWS services (Comprehend, Transcribe, Polly, data stored in S3, RDS, DDB, ...)
- commonly integrates with other services / apps / platforms

## Amazon Forecast 101

Forecasting for time-series data. Predicting, for example:

- retail demand
- supply chain
- staffing
- energy
- server capacity
- web traffic

Input: historical & related data. Output: forecast & forecast explainability.

You can interact with the product via Web Console (visualization), CLI, API, Python SDK.

## Amazon Fraud Detector

Fully managed fraud detection service.

Use cases:

- new account creations
- payments
- guest checkout

Upload historical data, choose model type:

- `Online Fraud`: little historical data, e.g. new customer account
- `Transaction Fraud`: transactional history, identifying suspect payments
- `Account Takeover`: identify phishing or another social based attack

All of the various events are scored; you can create rules (decision logic), which allow you to react to a score based on business activity.

## Amazon SageMaker

Collection of other products and features, packaged together into a fully managed machine learning service. Handles:

- Fetch
- Clean
- Prepare
- Train
- Evaluate
- Deploy
- Monitor/Collect

SageMaker `Studio` – IDE for ML lifecycle: build, train, debug and monitor models.

SageMaker `Domain` – isolation for a particular project: EFS Volume, Users, Apps, Policies, VPCs.

`Containers` – ML environments (OS, Libs, Tooling): Docker containers, deployed to ML EC2 instance.

`Hosting` – deploy endpoints for your models.

SageMaker has no cost, but the resources it creates – do (complex pricing).
