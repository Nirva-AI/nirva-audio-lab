# Vendor Library Local Changes

This document tracks all local modifications to vendored libraries.

---

## 3D-Speaker (CAM++ Speaker Verification)

**Status**: ⚠️ Modified  
**Commit**: `143c8ae` - "feat: Add CoreML conversion for CAM++ speaker verification"

### Added Files (17 total)

**CoreML Conversion:**
- `scripts/convert_campplus_to_coreml.py` - PyTorch → CoreML conversion
- `scripts/test_speaker_verification.py` - Testing and evaluation
- `campplus_cn_en_common.mlpackage/` - CoreML model (14MB, 192-dim)
- `campplus_cn_en_common.pt` - Downloaded PyTorch checkpoint

**Documentation:**
- `SPEAKER_VERIFICATION_COMPLETE.md` - Integration guide
- `LIBRARY_COMPARISON.md` - WhisperKit/FluidAudio comparison
- `PHASE1_COMPLETE_SUMMARY.md` - Conversion summary
- `iOS_INTEGRATION.md` - iOS patterns
- `FBANK_iOS_SOLUTIONS.md` - FBank extraction solutions

**Development Scripts:**
- Various inspection and validation scripts

### Model Details

- **Size**: 14 MB
- **Output**: 192-dim embeddings
- **Performance**: 0.65% EER on VoxCeleb1-O

---

## pyannote-audio (Speaker Diarization)

**Status**: ⚠️ Modified  
**Commit**: `be1e5bff` - "Add local API and audio samples for testing"

### Added Files (23 total)

**API Server:**
- `api/main.py` - FastAPI diarization server
- `api/requirements.txt`, `api/Dockerfile`
- Deployment scripts for Cloud Run and SageMaker

**Test Audio Samples:**
- 13 WAV files for testing (andy, han, jason, wei samples)

---

## Unmodified Libraries

- **FluidAudio**: ✅ Clean
- **WhisperKit**: ✅ Clean  
- **whisperkittools**: ✅ Clean

