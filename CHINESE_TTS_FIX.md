# 中文 TTS 支持修复

## 问题描述

Speaches 的 Piper 中文 TTS 模型无法正常工作，生成的音频为空（0 字节）。

## 根本原因

Docker 镜像中安装的 `piper-tts` 版本为 1.2.0，但代码要求 >= 1.3.0。

### API 差异

**旧版 API (1.2.0)**:
```python
voice.synthesize(text, wav_file, ...)  # 需要 wave.Wave_write 对象
```

**新版 API (>= 1.3.0)**:
```python
for audio_chunk in voice.synthesize(text, SynthesisConfig(...)):
    yield Audio(audio_chunk.audio_float_array, ...)
```

## 验证结果

### 音素化测试
中文文本可以正确转换为音素：
- "你好" → `['n', 'i', '2', 'χ', 'ˈ', 'ɑ', 'u', '2']` (8 个音素)
- "你好，这是一个测试。" → 46 个音素

### 使用旧版 API 测试
使用 piper-tts 1.2.0 的 API 可以成功生成中文音频：
- "你好" → 31,788 字节 WAV 文件
- "你好，这是一个测试。" → 72,236 字节 WAV 文件

## 解决方案

确保 Docker 镜像构建时使用 `uv.lock` 中锁定的 piper-tts 1.3.0 版本。

## 测试步骤

1. 构建新的 Docker 镜像
2. 下载中文模型：
   ```bash
   docker exec speaches uv tool run speaches-cli model download speaches-ai/piper-zh_CN-huayan-medium
   ```
3. 测试中文 TTS：
   ```bash
   curl -X POST http://localhost:8000/v1/audio/speech \
     -H "Content-Type: application/json" \
     -d '{
       "model": "speaches-ai/piper-zh_CN-huayan-medium",
       "input": "你好，这是一个中文语音合成测试。",
       "voice": "huayan",
       "response_format": "wav"
     }' \
     --output test_chinese.wav
   ```

## 可用的中文模型

- `speaches-ai/piper-zh_CN-huayan-medium` (推荐，22050 Hz)
- `speaches-ai/piper-zh_CN-huayan-x_low` (快速，16000 Hz)

## 相关文件

- `pyproject.toml`: 指定 `piper-tts>=1.3.0`
- `uv.lock`: 锁定 `piper-tts==1.3.0`
- `src/speaches/executors/piper.py`: 使用新版 API
