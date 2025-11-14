# Nirva On-Device Audio Processing Architecture

## 🎯 Complete System Overview

**Mission**: Process necklace recordings locally with 4 on-device AI tasks:
1. **ASR** (Automatic Speech Recognition) - Transcribe audio to text
2. **Speaker Diarization** - "Who spoke when?"
3. **Voiceprint Verification** - Identify if speaker is the user ("self")
4. **Emotion Detection** - Extract emotional state from speech
5. **Event Extraction** - Identify key moments/events

**Current Focus**: Tasks 1-3 (ASR + Diarization + Voiceprint)

---

## 📊 Current State Analysis

### Existing Components

#### ✅ Cloud-Based (nirva_service)
```python
# nirva_service/nirva/api/voiceprints.py
- Voiceprint enrollment via pyannote.ai
- Cloud-based speaker verification
- S3 storage for audio files
- Embedding extraction (cloud)
```

#### ✅ Flutter App (nirva_app)
```dart
// lib/services/voiceprint_service.dart
- Upload voiceprint audio to S3
- Check voiceprint status
- Download enrolled voiceprint
- Currently: ALL processing in cloud
```

#### ❌ Missing: On-Device Processing
- No local ASR
- No local diarization
- No local voiceprint comparison
- No model switcher (cloud vs on-device)

---

## 🏗️ Target Architecture

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    NECKLACE RECORDING                        │
│              Stored as audio file on device                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              PERIODIC ON-DEVICE PROCESSING                   │
│                  (Batch processing)                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
   ┌──────┐      ┌──────────┐    ┌──────────┐
   │ ASR  │      │Diarization│   │Voiceprint│
   │Model │      │  Model    │   │  Model   │
   └──┬───┘      └─────┬────┘    └────┬─────┘
      │                │              │
      ▼                ▼              ▼
   Transcript    Speaker Segments   User ID
                       │              │
                       └──────┬───────┘
                              ▼
                    ┌──────────────────┐
                    │ Replace speaker  │
                    │ with "self" tag  │
                    └─────────┬────────┘
                              ▼
                    ┌──────────────────┐
                    │ Diarized         │
                    │ Transcript       │
                    │ "Self: Hello..." │
                    │ "Person1: Hi..." │
                    └──────────────────┘
```

### Component Integration

```
nirva_app (Flutter)
├── ios/
│   ├── Runner/
│   │   ├── FluidAudioBridge.swift        # ASR + Diarization
│   │   ├── CAMPlusVoiceprintBridge.swift # NEW: Voiceprint matching
│   │   └── Resources/
│   │       ├── parakeet_tdt_coreml/      # ASR model
│   │       ├── segmentation.mlmodel       # Diarization
│   │       └── campplus_cn_en_common.mlpackage  # NEW: Voiceprint
│   │
├── lib/
│   ├── services/
│   │   ├── fluid_audio_service.dart       # ASR + Diarization
│   │   ├── voiceprint_verification_service.dart  # NEW: On-device voiceprint
│   │   ├── model_switcher_service.dart    # NEW: Cloud vs On-device
│   │   └── audio_processing_pipeline.dart # NEW: Orchestrator
│   │
│   └── pages/
│       ├── recordings_page.dart           # MODIFY: Show diarization
│       ├── recording_detail_page.dart     # NEW: Per-recording view
│       └── dev_recording_screen.dart      # NEW: Model switcher UI
│
└── 3D-Speaker/
    └── campplus_cn_en_common.mlpackage   # Ready to integrate ✅
```

---

## 🔄 Processing Pipeline (Detailed)

### Step 1: Offline ASR (Batch Processing)

**Trigger**: Periodically (e.g., every 30 min, or when charging)

```dart
// AudioProcessingPipeline.dart
Future<void> processRecording(String audioFilePath) async {
  // 1. Run ASR (FluidAudio)
  final transcript = await FluidAudioService.transcribe(audioFilePath);

  // transcript = [
  //   {text: "Hello there", start: 0.0, end: 2.1},
  //   {text: "How are you", start: 2.5, end: 4.0},
  // ]
}
```

**Models**:
- FluidAudio (Parakeet TDT v3)
- Output: Raw transcript with timestamps

---

### Step 2: Speaker Diarization

```dart
// AudioProcessingPipeline.dart (continued)
Future<void> processRecording(String audioFilePath) async {
  // ... ASR from Step 1

  // 2. Run Diarization (FluidAudio)
  final diarization = await FluidAudioService.diarize(audioFilePath);

  // diarization = [
  //   {speaker: "Speaker_0", start: 0.0, end: 2.5},
  //   {speaker: "Speaker_1", start: 2.5, end: 5.0},
  //   {speaker: "Speaker_0", start: 5.0, end: 7.0},
  // ]
}
```

**Models**:
- FluidAudio Segmentation
- FluidAudio WeSpeaker (256-dim embeddings)
- Output: Timeline with speaker labels (Speaker_0, Speaker_1, etc.)

---

### Step 3: Voiceprint Verification (NEW - CAM++)

**For each diarized speaker segment:**

```dart
// AudioProcessingPipeline.dart (continued)
Future<void> processRecording(String audioFilePath) async {
  // ... ASR + Diarization from Steps 1-2

  // 3. Load enrolled user voiceprint
  final userEmbedding = await VoiceprintService.getUserEmbedding();

  if (userEmbedding == null) {
    _logger.w('No enrolled voiceprint - skipping speaker verification');
    return diarization; // Return without "self" tagging
  }

  // 4. For each speaker, compare with user voiceprint
  final verifiedDiarization = [];

  for (final segment in diarization) {
    // Extract audio clip for this speaker segment
    final audioClip = extractAudioSegment(
      audioFilePath,
      start: segment.start,
      end: segment.end
    );

    // Extract embedding using CAM++
    final segmentEmbedding = await VoiceprintVerificationService
        .extractEmbedding(audioClip);

    // Compare with user's enrolled voiceprint
    final similarity = cosineSimilarity(userEmbedding, segmentEmbedding);

    // Threshold from EER evaluation (typically 0.5-0.6)
    final isUser = similarity > 0.52;

    verifiedDiarization.add({
      'speaker': isUser ? 'self' : segment.speaker,
      'start': segment.start,
      'end': segment.end,
      'confidence': similarity,
      'original_speaker': segment.speaker, // Keep for UI
    });
  }

  return verifiedDiarization;
}
```

**Models**:
- CAM++ (192-dim embeddings) ✅
- Cosine similarity comparison
- Output: Speaker labels with "self" tag

**Key Implementation:**

```swift
// ios/Runner/CAMPlusVoiceprintBridge.swift
class CAMPlusVoiceprintBridge: NSObject, FlutterPlugin {
    private let model: MLModel

    func extractEmbedding(audioData: [Float]) -> [Float] {
        // 1. Extract FBank features
        let fbank = extractFBank(audioData) // 80-dim

        // 2. Run CAM++ inference
        let input = campplus_cn_en_commonInput(
            fbank_features: fbank
        )
        let output = try! model.prediction(from: input)

        return Array(output.embedding) // 192-dim
    }

    func compareWithEnrolled(audioClip: [Float]) -> Double {
        let embedding = extractEmbedding(audioClip)
        let enrolledEmbedding = loadEnrolledEmbedding()

        return cosineSimilarity(embedding, enrolledEmbedding)
    }
}
```

---

### Step 4: Merge Transcript + Diarization

```dart
// AudioProcessingPipeline.dart (final)
Future<DiarizedTranscript> processRecording(String audioFilePath) async {
  final transcript = await FluidAudioService.transcribe(audioFilePath);
  final diarization = await diarizeWithVoiceprint(audioFilePath);

  // Merge: Assign transcript words to speakers
  final diarizedTranscript = mergeTranscriptAndDiarization(
    transcript,
    diarization
  );

  // Result:
  // [
  //   {speaker: "self", text: "Hello there", start: 0.0, end: 2.1},
  //   {speaker: "Speaker_1", text: "How are you", start: 2.5, end: 4.0},
  // ]

  return diarizedTranscript;
}
```

---

## 🎨 UI Requirements

### 1. Dev Recording Screen (Model Switcher)

**Purpose**: Easily toggle between cloud vs on-device models for testing

```dart
// lib/pages/dev_recording_screen.dart
class DevRecordingScreen extends StatefulWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Model Selection
        Card(
          child: Column(
            children: [
              Text('ASR Model'),
              SegmentedButton(
                segments: [
                  ButtonSegment(value: 'cloud', label: 'Cloud (Deepgram)'),
                  ButtonSegment(value: 'device', label: 'On-Device (Parakeet)'),
                ],
                selected: {_asrModel},
                onSelectionChanged: (Set<String> selection) {
                  setState(() => _asrModel = selection.first);
                },
              ),

              Text('Diarization Model'),
              SegmentedButton(
                segments: [
                  ButtonSegment(value: 'cloud', label: 'Cloud (pyannote.ai)'),
                  ButtonSegment(value: 'device', label: 'On-Device (FluidAudio)'),
                ],
                selected: {_diarizationModel},
                onSelectionChanged: (Set<String> selection) {
                  setState(() => _diarizationModel = selection.first);
                },
              ),

              Text('Voiceprint Model'),
              SegmentedButton(
                segments: [
                  ButtonSegment(value: 'cloud', label: 'Cloud (pyannote.ai)'),
                  ButtonSegment(value: 'device', label: 'On-Device (CAM++)'),
                ],
                selected: {_voiceprintModel},
                onSelectionChanged: (Set<String> selection) {
                  setState(() => _voiceprintModel = selection.first);
                },
              ),
            ],
          ),
        ),

        // Process Button
        ElevatedButton(
          onPressed: () => _processRecording(),
          child: Text('Process Recording'),
        ),

        // Results Display
        _buildResultsView(),
      ],
    );
  }

  Future<void> _processRecording() async {
    final result = await AudioProcessingPipeline.process(
      audioPath: widget.audioPath,
      config: ProcessingConfig(
        asrModel: _asrModel,
        diarizationModel: _diarizationModel,
        voiceprintModel: _voiceprintModel,
      ),
    );

    setState(() => _results = result);
  }
}
```

---

### 2. Recording Detail Page (Diarization View)

```dart
// lib/pages/recording_detail_page.dart
class RecordingDetailPage extends StatelessWidget {
  final DiarizedTranscript transcript;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: transcript.segments.length,
      itemBuilder: (context, index) {
        final segment = transcript.segments[index];

        return Card(
          color: segment.speaker == 'self'
              ? Colors.blue[50]   // Highlight user's speech
              : Colors.grey[100], // Other speakers
          child: ListTile(
            leading: CircleAvatar(
              child: Text(
                segment.speaker == 'self' ? '👤' : '🗣️'
              ),
            ),
            title: Text(segment.speaker),
            subtitle: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(segment.text),
                Text(
                  '${segment.start.toStringAsFixed(1)}s - ${segment.end.toStringAsFixed(1)}s',
                  style: TextStyle(fontSize: 12, color: Colors.grey),
                ),
              ],
            ),
            trailing: segment.speaker != 'self' && segment.speaker != 'unknown'
                ? ElevatedButton(
                    onPressed: () => _compareWithVoiceprint(segment),
                    child: Text('Compare'),
                  )
                : null,
          ),
        );
      },
    );
  }

  Future<void> _compareWithVoiceprint(Segment segment) async {
    // Extract audio clip for this segment
    final audioClip = await extractSegmentAudio(
      recording.audioPath,
      segment.start,
      segment.end,
    );

    // Compare with user's voiceprint
    final similarity = await VoiceprintVerificationService
        .compareWithEnrolled(audioClip);

    // Show result
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Voiceprint Match'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text('Similarity: ${(similarity * 100).toStringAsFixed(1)}%'),
            SizedBox(height: 16),
            LinearProgressIndicator(value: similarity),
            SizedBox(height: 16),
            Text(
              similarity > 0.52
                  ? '✅ Likely YOU'
                  : '❌ Different person',
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
                color: similarity > 0.52 ? Colors.green : Colors.red,
              ),
            ),
          ],
        ),
        actions: [
          if (similarity > 0.52)
            TextButton(
              onPressed: () => _markAsSelf(segment),
              child: Text('Mark as Self'),
            ),
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('Close'),
          ),
        ],
      ),
    );
  }
}
```

---

## 🔧 Implementation Plan

### Phase 1: CAM++ Integration (On-Device Voiceprint) ✅ COMPLETE

**Status**: Model converted, ready to integrate

- ✅ Convert CAM++ to CoreML (14MB, 192-dim)
- ✅ Create verification script (Python)
- ⏭️ Integrate into iOS app

### Phase 2: iOS Bridge (Week 1)

**Create Swift bridges:**

```swift
// CAMPlusVoiceprintBridge.swift (~200 lines)
- Load CAM++ CoreML model
- Extract FBank features
- Run embedding extraction
- Cosine similarity computation
- Caching enrolled embedding

// FlutterMethodChannel
- extractEmbedding(audioPath) -> [Float]
- compareWithEnrolled(audioPath) -> Double
- enrollVoiceprint(audioPath) -> [Float]
```

### Phase 3: Flutter Service Layer (Week 1-2)

```dart
// voiceprint_verification_service.dart
class VoiceprintVerificationService {
  // On-device methods
  static Future<List<double>> extractEmbedding(String audioPath)
  static Future<double> compareWithEnrolled(String audioPath)
  static Future<void> enrollVoiceprint(String audioPath)

  // Hybrid: Store embedding locally + cloud backup
  static Future<void> syncEmbeddingToCloud()
  static Future<List<double>?> getEnrolledEmbedding()
}

// model_switcher_service.dart
class ModelSwitcherService {
  ProcessingConfig config;

  Future<DiarizedTranscript> processRecording(String audioPath) {
    switch (config.voiceprintModel) {
      case 'cloud':
        return _processCloudVoiceprint(audioPath);
      case 'device':
        return _processDeviceVoiceprint(audioPath);
    }
  }
}

// audio_processing_pipeline.dart
class AudioProcessingPipeline {
  static Future<DiarizedTranscript> process({
    required String audioPath,
    required ProcessingConfig config,
  }) async {
    // 1. ASR
    final transcript = await _runASR(audioPath, config.asrModel);

    // 2. Diarization
    final diarization = await _runDiarization(audioPath, config.diarizationModel);

    // 3. Voiceprint verification
    final verified = await _verifyVoiceprints(
      audioPath,
      diarization,
      config.voiceprintModel
    );

    // 4. Merge
    return _mergeDiarizedTranscript(transcript, verified);
  }
}
```

### Phase 4: UI Implementation (Week 2)

1. **Dev Recording Screen**
   - Model switcher controls
   - Process button
   - Results comparison view

2. **Recording Detail Page**
   - Diarized transcript view
   - "Compare" button per speaker
   - Visual distinction for "self"

### Phase 5: Testing & Optimization (Week 3)

1. **Accuracy Testing**
   - Test on real necklace recordings
   - Tune similarity threshold
   - Measure false positive/negative rates

2. **Performance Optimization**
   - Batch processing optimization
   - Embedding caching
   - Background processing

3. **UX Refinement**
   - Loading states
   - Error handling
   - Offline support

---

## 📊 Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **ASR Latency** | < 2x real-time | 10 min audio → 20s processing |
| **Diarization** | < 3x real-time | Includes embedding extraction |
| **Voiceprint** | < 0.5s per segment | CAM++ inference |
| **Total Pipeline** | < 5x real-time | Full processing (60 min → 5 min) |
| **Voiceprint EER** | < 2% | False accept/reject rate |
| **Memory** | < 500MB peak | During processing |

---

## 🔐 Data Privacy

**On-Device Processing Benefits:**
- ✅ No audio uploaded to cloud (stays on device)
- ✅ Voiceprint embedding stored locally
- ✅ Optional cloud backup (encrypted)
- ✅ User controls all processing

**Hybrid Approach:**
- On-device: Default for all processing
- Cloud: Optional fallback for accuracy comparison
- User choice: Toggle in dev screen

---

## 📁 File Structure

```
nirva_app/
├── ios/Runner/
│   ├── CAMPlusVoiceprintBridge.swift          # NEW
│   ├── FluidAudioBridge.swift                 # MODIFY: Add diarization
│   ├── AudioSegmentExtractor.swift            # NEW: Extract clips
│   └── Resources/
│       └── campplus_cn_en_common.mlpackage    # NEW: 14MB
│
├── lib/
│   ├── services/
│   │   ├── voiceprint_verification_service.dart   # NEW
│   │   ├── model_switcher_service.dart            # NEW
│   │   ├── audio_processing_pipeline.dart         # NEW
│   │   └── fluid_audio_service.dart               # MODIFY
│   │
│   ├── models/
│   │   ├── diarized_transcript.dart               # NEW
│   │   ├── processing_config.dart                 # NEW
│   │   └── speaker_segment.dart                   # NEW
│   │
│   └── pages/
│       ├── dev_recording_screen.dart              # NEW
│       ├── recording_detail_page.dart             # NEW
│       └── recordings_page.dart                   # MODIFY
│
└── 3D-Speaker/
    ├── campplus_cn_en_common.mlpackage           # ✅ Ready
    └── scripts/
        └── test_speaker_verification.py          # ✅ Testing tool
```

---

## ✅ Next Steps

### Immediate (This Week):
1. Copy CAM++ model to iOS project
2. Create CAMPlusVoiceprintBridge.swift
3. Test embedding extraction on device
4. Create Flutter service wrapper

### Short-term (2-3 Weeks):
1. Implement dev recording screen
2. Build audio processing pipeline
3. Test on real recordings
4. Tune similarity threshold

### Medium-term (1-2 Months):
1. Optimize batch processing
2. Add emotion detection (text-based initially)
3. Implement event extraction
4. Production deployment

---

## 🎯 Success Criteria

1. ✅ On-device voiceprint verification working
2. ✅ "Self" vs "Other" labeling accurate (< 5% error)
3. ✅ Dev screen allows easy model switching
4. ✅ Processing completes in < 5x real-time
5. ✅ UI clearly shows diarization results
6. ✅ User can manually verify/correct speaker labels

---

**Ready to start implementation?**

The CAM++ model is ready, we have the architecture mapped out, and the integration path is clear. Let me know which phase you want to tackle first!
