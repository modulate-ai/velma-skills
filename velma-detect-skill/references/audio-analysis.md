# Audio analysis and fast transcription

Seven batch endpoints. All are `POST` with `multipart/form-data`, base `https://platform.modulate.ai`, file in `upload_file`, header `X-API-Key`, max 100 MB. All return JSON synchronously. Errors: see `errors-and-limits.md`.

"10 common extensions" below means `.aac .aiff .flac .mov .mp3 .mp4 .ogg .opus .wav .webm`. "Wide list" means the 96-extension list (wav, mp3, flac, m4a, ogg, opus, webm, aac, aiff, mp4, mov, amr, wma, mkv and more). Convert with ffmpeg if the extension is not accepted.

## Emotion detection

Use when the user wants the emotional tone of a recording over time, without a transcript.

Endpoint: `POST /api/velma-2-emotion-batch`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-emotion-batch \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@call.wav" > emotion.json
```

- Optional form fields: none.
- `emotion`: whole-file label. One of Angry, Disgusted, Afraid, Happy, Neutral, Other, Sad, Surprised, Unknown. Never null.
- `time_series[]`: consecutive fixed-length windows, each with `start_ms`, `duration_ms`, `emotion`. Empty for a file shorter than one window. A trailing remainder shorter than a window is dropped.
- Extensions: 10 common.

```bash
jq -r '"overall: \(.emotion)", (.time_series[] | "\(.start_ms/1000)s +\(.duration_ms/1000)s \(.emotion)")' emotion.json
```

Caveat: it labels vocal tone per window, not what the speaker feels. There is no per-speaker split. For a per-speaker emotion, use `emotion_signal=true` on transcription (`transcribe.md`).

## Accent detection

Use when the user wants the accent of the speaker(s) over time.

Endpoint: `POST /api/velma-2-accent-batch`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-accent-batch \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@speaker.wav" > accent.json
```

Optional form fields:
- `use_ensemble` (boolean, default false): more thorough analysis. It may change the returned labels and adds latency.
- `training_permitted` (boolean, default true): whether this request's audio may be used to improve the models. Set `-F "training_permitted=false"` when the audio is sensitive and the user wants to opt out.

Response:
- `accent`: whole-file label, a free string such as `British`. Never null.
- `time_series[]`: fixed-length windows with `start_ms`, `duration_ms`, `accent`. Empty for a file shorter than one window.
- Extensions: 10 common.

Caveat: the docs list no fixed label set, so report the label as returned. An accent label is not a statement about nationality, origin or identity. Do not use it to profile a person.

## Language detection

Use when the user asks which language is spoken, or needs to route audio by language. No transcript.

Endpoint: `POST /api/velma-2-language-detection-batch`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-language-detection-batch \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@recording.mp3" > language.json
```

- Optional form fields: none.
- `predicted_language`: display name, for example `English`.
- `predicted_language_code`: lowercase ISO 639-1, for example `en`.
- `confidence`: 0 to 1 probability for that language.
- `duration_ms`: decoded audio length.
- Extensions: wide list.

```bash
jq '{predicted_language, predicted_language_code, confidence}' language.json
```

Caveat: only the first 30 seconds are analysed, whatever the file length, and the result is one language. For mixed-language audio it says nothing about the later parts. The docs give no threshold; a low `confidence` (the docs show 0.41 as an example) means treat the label as a guess.

## Audio event detection

Use when the user asks whether specific non-speech sounds occur: crying, laughter, applause, gunshot, glass shattering, dog bark, cough, knock, telephone ring, instruments and similar.

Endpoint: `POST /api/velma-2-audio-event-classifier`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-audio-event-classifier \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@clip.wav" > events.json
```

- Optional form fields: none.
- `probs`: object of event name to probability (0 to 1). Only events detected with enough confidence are present. Events are not mutually exclusive and the values do not sum to 1. An empty object means nothing was confidently detected.
- `duration_ms`: audio length.
- Event names (fixed list): Crying, Acoustic_guitar, Applause, Bark, Bass_drum, Burping_or_eructation, Bus, Cello, Chime, Clarinet, Computer_keyboard, Cough, Cowbell, Double_bass, Drawer_open_or_close, Electric_piano, Fart, Finger_snapping, Fireworks, Flute, Glockenspiel, Gong, Gunshot_or_gunfire, Harmonica, Hi_hat, Keys_jangling, Knock, Laughter, Meow, Microwave_oven, Oboe, Saxophone, Scissors, Shatter, Snare_drum, Squeak, Tambourine, Tearing, Telephone, Trumpet, Violin_or_fiddle, Writing.
- Extensions: 10 common.

```bash
jq -r '.probs | to_entries | sort_by(-.value)[] | "\(.key)\t\(.value)"' events.json
```

Caveat: clip-level only, with no timestamps. An absent event is not proof it is absent from the audio. The list is fixed: there is no alarm, siren or smoke-detector class, so say so when the user asks for one.

## Music and speech detection

Use when the user wants to know whether a clip contains music, speech, or both, and where. This is not AI-music detection; for "is this song AI-made?" use `ai-music-detect.md`.

Endpoint: `POST /api/velma-2-music-detection-batch`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-music-detection-batch \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@broadcast.wav" > musicspeech.json
```

- Optional form fields: none.
- `primary_label`: `music`, `speech`, `neither` or `unknown` (no frames could be produced).
- `music_pct`, `speech_pct`: 0 to 100, share of the clip classified as each.
- `frames[]`: about 192 ms each, contiguous, with `start_time_ms`, `end_time_ms`, `music_prob`, `speech_prob`. Music and speech are not exclusive: both can be high in one frame.
- `duration_s`, `latency_ms`, `filename`.
- Extensions: wide list.

Time ranges where music probability is above 0.5 (0.5 is an arbitrary cut, not an API value), merged:

```bash
jq -r '[.frames[] | select(.music_prob > 0.5)] | reduce .[] as $f ([]; if length>0 and .[-1][1]==$f.start_time_ms then .[-1][1]=$f.end_time_ms else .+[[$f.start_time_ms,$f.end_time_ms]] end) | .[] | "\(.[0]/1000)s-\(.[1]/1000)s"' musicspeech.json
```

Caveat: probabilities per frame, not proof. The response can be large for long audio (about 5 frames per second); summarise rather than print it.

## English fast transcription

Use for quick English-only transcripts, optionally with word times and speaker turns. It returns no emotion, accent or PII tagging.

Endpoint: `POST /api/velma-2-stt-batch-english-vfast`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-stt-batch-english-vfast \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@call.wav" \
  -F "speaker_diarization=true" \
  -F "time_stamps=true" > transcript.json
```

Optional form fields (all boolean, all default false):

| Field | Effect |
| - | - |
| `speaker_diarization` | Adds `utterances[]` of speaker turns. `text` becomes the concatenation of the turns. Slower than the default path, and the transcript can differ from the same audio sent without it. Quiet background speech can appear as extra utterances. Overlapping speech goes to one speaker per time span |
| `time_stamps` | Adds `words[]` with `word`, `start`, `end` (seconds) and `timing`. Without diarization it adds latency that grows with audio length |
| `medical_domain` | For English medical dictation. Corrects misheard clinical terms (drug names, conditions, procedures) with a medical language model and tries to recover an opening or ending the transcriber dropped, such as patient name, chart number or date of birth. Adds some latency. The docs give no accuracy figure |

Response:
- `text`, `duration_ms`. With both flags off, that is all. Optional keys are absent, never null.
- `utterances[]` (diarization): `utterance_uuid`, `text`, `start_ms`, `duration_ms`, `speaker` (1-indexed). With `time_stamps` also on, each utterance has `words[]`.
- `words[]` (timestamps): `timing` is `aligned`, `partial` or `estimated`. `confidence` scores how well the word was placed in the audio, not whether the word is right. It is absent on `estimated` words. Times are seconds from the file start, while utterance times are milliseconds.
- `words_unavailable` replaces `words` when timings could not be produced. Branch on `reason`: `audio_too_long_for_alignment`, `alignment_disabled`, `alignment_failed` (retryable), `empty_transcript`. The transcript is unaffected. `speaker_diarization=true` has no length ceiling for word times. Read `max_audio_seconds` from the response, do not hard-code it.
- Extensions: the spec text lists both the wide list and the 10 common ones, and the 400 message shows the wide list. A live check accepted an extension from the wide list (`.aif`), so either list works. Prefer the 10 common extensions if you are unsure.

```bash
jq -r '.utterances[] | "[\(.start_ms / 1000 | floor)s] speaker \(.speaker): \(.text)"' transcript.json
```

Caveat: English only. For other languages use multilingual fast transcription. For redaction, emotion, accent or custom vocabulary use `transcribe.md`. Medical output still needs human review before clinical use.

## Multilingual fast transcription

Use for quick transcripts in any supported language, or when the language is unknown.

Endpoint: `POST /api/velma-2-stt-batch-multilingual-vfast`

```bash
curl -sS -X POST https://platform.modulate.ai/api/velma-2-stt-batch-multilingual-vfast \
  -H "X-API-Key: $VELMA_API_KEY" \
  -F "upload_file=@reunion.mp3" \
  -F "language=fr" > transcript.json
```

- Optional form field: `language`, a short code such as `en`, `es`, `fr`, `ja`. Declaring it takes the fastest path and is echoed back. Omit it to auto-detect.
- Response: `text`, `duration_ms`, `language`. No diarization, utterances, emotion, accent or PII tagging.
- `language`: the declared value echoed unchanged, or the detected code. For mixed-language audio it is the predominant language; the transcript still follows each part as spoken.
- Extensions: 10 common.

Caveat: the docs say a declared `language` is transcribed directly in that language. If unsure of the language, omit the field. For speaker labels use `/api/velma-2-stt-batch` (`transcribe.md`).
