# CaptionLab

CaptionLab is an Android app for generating image captions and comparing on-device and cloud-based vision models. It includes an interactive captioning screen, batch benchmarks, performance measurements, and CSV/JSON export.

The project is useful for experimenting with image-captioning models on Android and studying the trade-offs between local inference and hosted APIs.

## Features

- Generate a caption for an image selected from the device or captured with the camera.
- Compare local ONNX models with cloud providers through a shared interface.
- Run batch benchmarks across image sets, with warm-up runs and configurable repetitions.
- Track preprocessing, inference, postprocessing, and end-to-end time, along with memory, energy, and API cost metrics where available.
- Export benchmark results as CSV and JSON, plus a text summary.
- Review benchmark progress and results in the app.

## Supported providers

| Provider | Model or service | Runs on |
| --- | --- | --- |
| Florence-2 | Florence-2-base | Device (ONNX Runtime) |
| ViT-GPT2 | ViT-GPT2 image captioning | Device (ONNX Runtime) |
| BLIP | BLIP image-captioning-base | Device (ONNX Runtime) |
| OpenAI | GPT-4o-mini | Cloud API |
| Azure | Computer Vision Image Analysis | Cloud API |
| Google | Gemini 2.5 Flash Lite | Cloud API |

Cloud provider names and pricing can change over time. Refer to each provider's current documentation for API availability and pricing.

## Requirements

- Android Studio with Android SDK 36 installed.
- JDK 17 for the Gradle build.
- An Android device or emulator running Android 8.0 (API 26) or newer.
- Internet access and an API key to use cloud providers.

The app packages local model files from `Application/app/src/main/assets/models/`. These model assets can make the resulting install/build large.

## Build and run

1. Clone this repository and open the `Application` directory in Android Studio.
2. Let Android Studio sync the Gradle project and install any required Android SDK components.
3. Run the `app` configuration on a connected device or emulator.

To build from the command line from the `Application` directory:

```bash
# macOS or Linux
./gradlew assembleDebug

# Windows
gradlew.bat assembleDebug
```

The debug APK is written to `Application/app/build/outputs/apk/debug/`.

## Using cloud providers

Add API keys in the app's settings for the providers you want to use. Requests to cloud providers send the selected image to the corresponding external service. Local providers run inference on the device.

For development convenience, the app stores entered keys in its local preferences. Do not use production credentials in a shared or untrusted device, and do not commit API keys to the repository. Review each provider's terms and data-handling policy before sending images.

## Benchmarking and exports

The single-image screen is intended for interactive testing. The batch benchmark screen lets you select image sets, choose providers, configure warm-up and measurement runs, and monitor progress.

Benchmark exports include per-run results and, where supported by the provider, aggregated statistics. CSV is suitable for spreadsheet and data-analysis tools; JSON includes structured experiment and device metadata. The app also writes a plain-text summary.

Exports are stored in the app-specific external files directory under `exports` (or the app's internal files directory when external storage is unavailable). They are not written to a shared public folder automatically.

Measurements depend on the Android device, operating system, network, provider service, and model runtime. Energy and memory figures should be treated as estimates for comparison under a given test setup rather than universal hardware-independent values. Cloud costs depend on current provider pricing and usage.

## Architecture

CaptionLab uses a common `CaptioningProvider` interface so the app can invoke local models and cloud APIs through the same flow. `ProviderManager` registers providers, `MetricsCollector` records measurements, `BenchmarkRunner` coordinates batch runs, and `DataExporter` writes the results.

```
                     ╔════════════════════════════════════════╗
                     ║     CaptionLab Android Application     ║
                     ╚════════════════════════════════════════╝
                                         │
                 ┌───────────────────────┼───────────────────────┐
                 │                       │                       │
                 ▼                       ▼                       ▼
        ┌────────────────┐     ┌─────────────────┐    ┌─────────────────┐
        │  Presentation  │     │    Benchmark    │    │   Data Export   │
        │     Layer      │     │     System      │    │     System      │
        ├────────────────┤     ├─────────────────┤    ├─────────────────┤
        │  MainActivity  │     │ BenchmarkRunner │    │  DataExporter   │
        │BatchTestActivity     │ MetricsCollector│    │  (CSV/JSON)     │
        │                │     │ ProgressListener│    │                 │
        └────────┬───────┘     └────────┬────────┘    └────────┬────────┘
                 │                      │                      │
                 └──────────────────────┼──────────────────────┘
                                        │
                                        ▼
                        ┌───────────────────────────────┐
                        │     ProviderManager           │
                        │  (Strategy Pattern Registry)  │
                        │                               │
                        │  • Provider Registration      │
                        │  • API Key Management         │
                        │  • Local/Cloud Categorization │
                        └───────────────┬───────────────┘
                                        │
                        ┌───────────────┴───────────────┐
                        │ CaptioningProvider (Interface)│
                        │                               │
                        │  + id: String                 │
                        │  + caption(Bitmap): Result    │
                        └───────────────┬───────────────┘
                                        │
                ┌───────────────────────┴───────────────────────┐
                │                                               │
                ▼                                               ▼
    ┌───────────────────────┐                   ┌──────────────────────────┐
    │   Local Providers     │                   │    Cloud Providers       │
    │    (ONNX Runtime)     │                   │     (REST APIs)          │
    ├───────────────────────┤                   ├──────────────────────────┤
    │ • Florence2Provider   │                   │ • OpenAIProvider         │
    │   (~215 MB)           │                   │   (GPT-4o-mini)          │
    │                       │                   │                          │
    │ • ViTGPT2Provider     │                   │ • AzureVisionProvider    │
    │   (~387 MB)           │                   │   (Florence-based API)   │
    │                       │                   │                          │
    │ • BlipProvider        │                   │ • GeminiProvider         │
    │   (~241 MB)           │                   │   (Gemini Pro Vision)    │
    └───────────┬───────────┘                   └───────────┬──────────────┘
                │                                           │
                ▼                                           ▼
    ┌───────────────────────┐                   ┌──────────────────────────┐
    │  ONNX Runtime Engine  │                   │   OkHttp HTTP Client     │
    ├───────────────────────┤                   ├──────────────────────────┤
    │ • Session Management  │                   │ • REST API Calls         │
    │ • Tensor Operations   │                   │ • JSON Serialization     │
    │ • Model Inference     │                   │ • Response Parsing       │
    │ • Memory Management   │                   │ • Error Handling         │
    └───────────────────────┘                   └──────────────────────────┘
                │                                           │
                └───────────────────┬───────────────────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │   Storage & Assets    │
                        ├───────────────────────┤
                        │ • ONNX Models (684MB) │
                        │   - Florence-2: 213MB │
                        │   - ViT-GPT2: 232MB   │
                        │   - BLIP: 239MB       │
                        │ • Tokenizer Configs   │
                        │ • Test Datasets       │
                        │ • Exported Results    │
                        └───────────────────────┘
```

The Android app is written in Kotlin and uses AndroidX, Material Components, Kotlin Coroutines, ONNX Runtime, OkHttp, and Moshi.

## Models and references

- [Florence-2-base](https://huggingface.co/microsoft/Florence-2-base)
- [ViT-GPT2 image captioning](https://huggingface.co/nlpconnect/vit-gpt2-image-captioning)
- [BLIP image-captioning-base](https://huggingface.co/Salesforce/blip-image-captioning-base)
- [ONNX Runtime for Android](https://onnxruntime.ai/docs/get-started/with-java.html)
- [OpenAI vision guide](https://platform.openai.com/docs/guides/images-vision)
- [Azure AI Vision documentation](https://learn.microsoft.com/azure/ai-services/computer-vision/)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)

## Project structure

```text
Application/   Android application and Gradle build
data/          Benchmark datasets, results, and analysis scripts
```

## Contributing

Contributions, bug reports, and suggestions are welcome. When reporting benchmark results, include the device model, Android version, selected model/provider, and benchmark configuration so the results can be interpreted in context.
