package com.freemotion.studio

import android.annotation.SuppressLint
import android.content.ContentValues
import android.content.Intent
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.os.Environment
import android.provider.MediaStore
import android.speech.tts.TextToSpeech
import android.util.Base64
import android.webkit.JavascriptInterface
import android.webkit.ValueCallback
import android.webkit.WebChromeClient
import android.webkit.WebSettings
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.Toast
import androidx.activity.OnBackPressedCallback
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import java.io.File
import java.io.FileOutputStream
import java.util.Locale

class MainActivity : AppCompatActivity(), TextToSpeech.OnInitListener {

    private lateinit var webView: WebView
    private var fileUploadCallback: ValueCallback<Array<Uri>>? = null
    private var tts: TextToSpeech? = null
    private var ttsReady = false

    private val fileChooserLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        val results: Array<Uri>? = if (result.resultCode == RESULT_OK) {
            result.data?.data?.let { arrayOf(it) }
        } else {
            null
        }
        fileUploadCallback?.onReceiveValue(results)
        fileUploadCallback = null
    }

    @SuppressLint("SetJavaScriptEnabled")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        tts = TextToSpeech(this, this)

        webView = WebView(this)
        setContentView(webView)

        webView.settings.apply {
            javaScriptEnabled = true
            domStorageEnabled = true
            allowFileAccess = true
            allowContentAccess = true
            mediaPlaybackRequiresUserGesture = false
            mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
            cacheMode = WebSettings.LOAD_DEFAULT
        }

        webView.addJavascriptInterface(AndroidVideoBridge(), "AndroidBridge")

        webView.webViewClient = WebViewClient()
        webView.webChromeClient = object : WebChromeClient() {
            override fun onShowFileChooser(
                webView: WebView?,
                filePathCallback: ValueCallback<Array<Uri>>?,
                fileChooserParams: FileChooserParams?
            ): Boolean {
                fileUploadCallback?.onReceiveValue(null)
                fileUploadCallback = filePathCallback

                val intent = Intent(Intent.ACTION_GET_CONTENT).apply {
                    addCategory(Intent.CATEGORY_OPENABLE)
                    type = "image/*"
                }
                fileChooserLauncher.launch(Intent.createChooser(intent, "Select Image"))
                return true
            }
        }

        webView.loadUrl("file:///android_asset/index.html")

        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                if (webView.canGoBack()) {
                    webView.goBack()
                } else {
                    finish()
                }
            }
        })
    }

    override fun onInit(status: Int) {
        if (status == TextToSpeech.SUCCESS) {
            val res = tts?.setLanguage(Locale.US)
            ttsReady = res != TextToSpeech.LANG_MISSING_DATA && res != TextToSpeech.LANG_NOT_SUPPORTED
        }
    }

    override fun onDestroy() {
        tts?.stop()
        tts?.shutdown()
        super.onDestroy()
    }

    inner class AndroidVideoBridge {

        @JavascriptInterface
        fun speakText(text: String) {
            if (ttsReady && text.isNotBlank()) {
                tts?.speak(text, TextToSpeech.QUEUE_FLUSH, null, "scene_narration")
            }
        }

        @JavascriptInterface
        fun stopSpeech() {
            tts?.stop()
        }

        @JavascriptInterface
        fun saveBase64Video(base64Data: String, fileName: String, mimeType: String): String {
            return try {
                val cleanBase64 = base64Data.substringAfter(",")
                val bytes = Base64.decode(cleanBase64, Base64.DEFAULT)

                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                    val values = ContentValues().apply {
                        put(MediaStore.Video.Media.DISPLAY_NAME, fileName)
                        put(MediaStore.Video.Media.MIME_TYPE, mimeType)
                        put(
                            MediaStore.Video.Media.RELATIVE_PATH,
                            "${Environment.DIRECTORY_MOVIES}/FreeMotion"
                        )
                        put(MediaStore.Video.Media.IS_PENDING, 1)
                    }

                    val resolver = contentResolver
                    val uri = resolver.insert(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, values)
                        ?: return "ERROR: Could not create MediaStore entry"

                    resolver.openOutputStream(uri)?.use { out ->
                        out.write(bytes)
                    }

                    values.clear()
                    values.put(MediaStore.Video.Media.IS_PENDING, 0)
                    resolver.update(uri, values, null, null)
                } else {
                    val moviesDir = File(
                        Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_MOVIES),
                        "FreeMotion"
                    )
                    if (!moviesDir.exists()) moviesDir.mkdirs()
                    val outFile = File(moviesDir, fileName)
                    FileOutputStream(outFile).use { it.write(bytes) }
                }

      <!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1">
<title>FreeMotion AI Studio</title>
<style>
  :root {
    --bg: #0b0f1d;
    --panel: #151c32;
    --muted: #98a6c7;
    --accent: #8b7cff;
    --cyan: #38e2c6;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    background: radial-gradient(circle at 85% 0%, #232152 0%, #0b0f1d 52%);
    color: #f6f7ff;
    font: 15px system-ui, -apple-system, sans-serif;
  }
  header {
    padding: 18px 16px 10px;
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .logo {
    width: 42px;
    height: 42px;
    border-radius: 13px;
    background: linear-gradient(135deg, var(--accent), var(--cyan));
    display: grid;
    place-items: center;
    font-size: 20px;
    font-weight: 900;
    color: #0b0f1d;
  }
  h1 { font-size: 19px; margin: 0; }
  header small { color: var(--muted); display: block; margin-top: 2px; }
  main { padding: 6px 14px 32px; max-width: 720px; margin: auto; }
  .tabs { display: flex; gap: 8px; margin: 10px 0 14px; }
  .tab {
    flex: 1;
    padding: 11px 8px;
    border: 1px solid #2e3856;
    border-radius: 12px;
    background: #141a2e;
    color: #c6d0ea;
    font-weight: 700;
    font-size: 13px;
  }
  .tab.active {
    background: #312c66;
    border-color: var(--accent);
    color: #fff;
  }
  .panel {
    background: rgba(21, 28, 50, 0.96);
    border: 1px solid #2e3856;
    border-radius: 16px;
    padding: 15px;
    margin-bottom: 14px;
  }
  label { display: block; font-weight: 700; margin: 10px 0 6px; font-size: 13px; }
  textarea, input, select {
    width: 100%;
    border: 1px solid #384466;
    background: #0d1324;
    color: #fff;
    border-radius: 11px;
    padding: 11px;
    font: inherit;
  }
  textarea { min-height: 92px; resize: vertical; }
  .row { display: flex; gap: 10px; }
  .row > * { flex: 1; }
  button.primary {
    width: 100%;
    border: 0;
    border-radius: 13px;
    padding: 14px;
    background: linear-gradient(100deg, var(--accent), #5299ff);
    color: #fff;
    font-size: 15px;
    font-weight: 800;
    margin-top: 14px;
  }
  button.secondary {
    width: 100%;
    border: 1px solid var(--cyan);
    border-radius: 13px;
    padding: 13px;
    background: rgba(56, 226, 198, 0.14);
    color: var(--cyan);
    font-size: 15px;
    font-weight: 800;
    margin-top: 12px;
  }
  button:disabled { opacity: 0.45; }
  .hint { color: var(--muted); font-size: 12px; line-height: 1.45; margin-top: 8px; }
  .chip {
    display: inline-block;
    background: #212a46;
    color: #cdd6f4;
    padding: 4px 9px;
    border-radius: 20px;
    font-size: 11px;
    margin: 2px 4px 4px 0;
  }
  .preview {
    background: #060911;
    border-radius: 13px;
    overflow: hidden;
    display: flex;
    justify-content: center;
  }
  .preview canvas {
    max-height: 430px;
    max-width: 100%;
    display: block;
  }
  .progress {
    height: 7px;
    background: #27304b;
    border-radius: 9px;
    overflow: hidden;
    margin-top: 12px;
  }
  .bar {
    height: 100%;
    width: 0;
    background: linear-gradient(90deg, var(--accent), var(--cyan));
    transition: width 0.15s linear;
  }
  #status { color: var(--cyan); margin-top: 9px; min-height: 20px; font-size: 13px; }
  .hidden { display: none; }
</style>
</head>
<body>

<header>
  <div class="logo">▶</div>
  <div>
    <h1>FreeMotion AI Studio</h1>
    <small>Free AI Shorts &amp; Motion Video Maker</small>
  </div>
</header>

<main>
  <div class="tabs">
    <button class="tab active" id="tabAi" onclick="switchMode('ai')">✦ AI Short Video</button>
    <button class="tab" id="tabPhoto" onclick="switchMode('photo')">▧ Photo Motion</button>
  </div>

  <section class="panel">
    <div id="aiMode">
      <label>Video Idea / Story Prompt</label>
      <textarea id="prompt" placeholder="Example: A neon cyberpunk cat rides a hoverboard through Tokyo at night. Rain reflects colorful signs. It jumps over a glowing rooftop."></textarea>

      <div class="row">
        <div>
          <label>Format</label>
          <select id="aspect" onchange="updateCanvasSize()">
            <option value="9:16" selected>9:16 (Shorts / Reels)</option>
            <option value="16:9">16:9 (YouTube)</option>
          </select>
        </div>
        <div>
          <label>Visual Style</label>
          <select id="style">
            <option value="cinematic 3d render, dramatic lighting">Cinematic 3D</option>
            <option value="anime illustration, studio ghibli vibrant colors">Anime</option>
            <option value="pixar style cute 3d cartoon">Kids Cartoon</option>
            <option value="cyberpunk neon digital art">Neon Cyberpunk</option>
          </select>
        </div>
      </div>

      <div class="row">
        <div>
          <label>Duration</label>
          <select id="duration">
            <option value="9">9 sec (3 scenes)</option>
            <option value="12" selected>12 sec (3 scenes)</option>
            <option value="15">15 sec (3 scenes)</option>
          </select>
        </div>
        <div>
          <label>AI Voiceover (Preview)</label>
          <select id="voice">
            <option value="on" selected>On (Android TTS)</option>
            <option value="off">Muted</option>
          </select>
        </div>
      </div>
    </div>

    <div id="photoMode" class="hidden">
      <label>Choose a Picture</label>
      <input type="file" id="photoInput" accept="image/*">
      <label>Caption Overlay (Optional)</label>
      <textarea id="caption" placeholder="Morning mist over the northern mountains..."></textarea>
      <div class="row">
        <div>
          <label>Camera Motion</label>
          <select id="motion">
            <option value="zoom">Slow Zoom In</option>
            <option value="out">Slow Zoom Out</option>
            <option value="pan">Pan Across</option>
            <option value="float">Gentle Float</option>
          </select>
        </div>
        <div>
          <label>Duration</label>
          <select id="imgDuration">
            <option value="6">6 sec</option>
            <option value="10" selected>10 sec</option>
            <option value="15">15 sec</option>
          </select>
        </div>
      </div>
    </div>

    <div class="hint" style="margin-top:12px;">
      <span class="chip">100% Free</span>
      <span class="chip">No API Key Needed</span>
      <span class="chip">Saves to Movies/FreeMotion</span>
    </div>

    <button class="primary" id="makeBtn" onclick="startGeneration()">Generate Free AI Video</button>
    <div class="progress"><div class="bar" id="bar"></div></div>
    <div id="status">Ready to create.</div>
  </section>

  <section class="panel">
    <label>Live Preview</label>
    <div class="preview">
      <canvas id="stage" width="720" height="1280"></canvas>
    </div>
    <button class="secondary hidden" id="saveBtn" onclick="saveToGallery()">⬇ Save Video to Phone Gallery</button>
    <p class="hint">Videos are saved directly to your phone's <b>Movies/FreeMotion</b> folder.</p>
  </section>
</main>

<script>
const canvas = document.getElementById('stage');
const ctx = canvas.getContext('2d');
let mode = 'ai';
let uploadedImg = null;
let sceneImages = [];
let sceneTexts = [];
let lastVideoBlob = null;
let lastMimeType = 'video/webm';

function switchMode(m) {
  mode = m;
  document.getElementById('aiMode').classList.toggle('hidden', m !== 'ai');
  document.getElementById('photoMode').classList.toggle('hidden', m !== 'photo');
  document.getElementById('tabAi').classList.toggle('active', m === 'ai');
  document.getElementById('tabPhoto').classList.toggle('active', m === 'photo');
  updateCanvasSize();
}

function updateCanvasSize() {
  const aspect = mode === 'ai' ? document.getElementById('aspect').value : '16:9';
  if (aspect === '9:16') {
    canvas.width = 720;
    canvas.height = 1280;
  } else {
    canvas.width = 1280;
    canvas.height = 720;
  }
  drawPlaceholder();
}

function drawPlaceholder() {
  const w = canvas.width, h = canvas.height;
  const grad = ctx.createLinearGradient(0, 0, w, h);
  grad.addColorStop(0, '#121936');
  grad.addColorStop(1, '#261e4d');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, w, h);

  ctx.fillStyle = '#ffffff';
  ctx.textAlign = 'center';
  ctx.font = '800 36px system-ui';
  ctx.fillText('FREEMOTION AI', w / 2, h / 2 - 14);
  ctx.fillStyle = '#9eb0d9';
  ctx.font = '22px system-ui';
  ctx.fillText('Tap "Generate Free AI Video" above', w / 2, h / 2 + 26);
}

document.getElementById('photoInput').addEventListener('change', (e) => {
  const f = e.target.files[0];
  if (!f) return;
  uploadedImg = new Image();
  uploadedImg.onload = () => drawKenBurnsFrame(uploadedImg, 0, 10, 'zoom', document.getElementById('caption').value);
  uploadedImg.src = URL.createObjectURL(f);
});

function splitIntoThreeScenes(promptText) {
  const clean = promptText.trim() || 'A glowing crystal forest at twilight. Magical fireflies dance in the air. A serene waterfall shines under the stars.';
  let parts = clean.split(/[.!?]+/).map(s => s.trim()).filter(Boolean);
  if (parts.length === 1) {
    parts = [
      parts[0],
      parts[0] + ', cinematic wide angle',
      parts[0] + ', dramatic close up'
    ];
  } else if (parts.length === 2) {
    parts.push(parts[0] + ', grand finale');
  }
  return parts.slice(0, 3);
}

function loadAiSceneImage(scenePrompt, style, w, h, seed) {
  return new Promise((resolve) => {
    const img = new Image();
    img.crossOrigin = 'anonymous';
    const fullPrompt = encodeURIComponent(`${scenePrompt}, ${style}, high detail, no text, no watermark`);
    const url = `https://image.pollinations.ai/prompt/${fullPrompt}?width=${w}&height=${h}&seed=${seed}&nologo=true`;

    const timer = setTimeout(() => resolve(null), 18000);
    img.onload = () => { clearTimeout(timer); resolve(img); };
    img.onerror = () => { clearTimeout(timer); resolve(null); };
    img.src = url;
  });
}

function wrapText(text, maxWidth) {
  const words = text.split(/\s+/);
  const lines = [];
  let line = '';
  for (const word of words) {
    const test = line ? line + ' ' + word : word;
    if (ctx.measureText(test).width > maxWidth && line) {
      lines.push(line);
      line = word;
    } else {
      line = test;
    }
  }
  if (line) lines.push(line);
  return lines;
}

function drawCaptionBox(text) {
  if (!text) return;
  const w = canvas.width, h = canvas.height;
  ctx.font = '800 34px system-ui';
  const lines = wrapText(text, w * 0.84).slice(0, 3);
  const boxH = lines.length * 44 + 34;
  const boxY = h - boxH - (h * 0.08);

  ctx.fillStyle = 'rgba(8, 12, 24, 0.72)';
  ctx.beginPath();
  ctx.roundRect(w * 0.06, boxY, w * 0.88, boxH, 18);
  ctx.fill();

  ctx.fillStyle = '#ffffff';
  ctx.textAlign = 'center';
  lines.forEach((line, idx) => {
    ctx.fillText(line, w / 2, boxY + 46 + idx * 44);
  });
}

function drawKenBurnsFrame(img, t, duration, motionType, caption) {
  const w = canvas.width, h = canvas.height;
  ctx.fillStyle = '#070a12';
  ctx.fillRect(0, 0, w, h);

  const progress = Math.min(1, Math.max(0, t / duration));
  let scale = 1.05;
  if (motionType === 'zoom') scale = 1.05 + progress * 0.22;
  if (motionType === 'out') scale = 1.27 - progress * 0.22;

  if (img && img.width > 0) {
    const iw = img.width, ih = img.height;
    const base = Math.max(w / iw, h / ih);
    const s = base * scale;
    const sw = w / s, sh = h / s;
    let sx = (iw - sw) / 2;
    let sy = (ih - sh) / 2;

    if (motionType === 'pan') sx = Math.max(0, (iw - sw) * progress);
    if (motionType === 'float') {
      sx = Math.max(0, (iw - sw) / 2 + Math.sin(t * 1.4) * iw * 0.02);
      sy = Math.max(0, (ih - sh) / 2 + Math.cos(t * 1.1) * ih * 0.02);
    }
    ctx.drawImage(img, sx, sy, sw, sh, 0, 0, w, h);
  } else {
    // Fallback gradient background if network image timed out
    const grad = ctx.createLinearGradient(0, 0, w, h);
    grad.addColorStop(0, '#191d42');
    grad.addColorStop(1, '#3b2466');
    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, w, h);
  }

  drawCaptionBox(caption);
}

async function startGeneration() {
  const makeBtn = document.getElementById('makeBtn');
  const saveBtn = document.getElementById('saveBtn');
  const statusEl = document.getElementById('status');
  const barEl = document.getElementById('bar');

  if (mode === 'photo' && !uploadedImg) {
    alert('Please choose a photo first.');
    return;
  }

  makeBtn.disabled = true;
  saveBtn.classList.add('hidden');
  barEl.style.width = '5%';

  if (mode === 'ai') {
    const promptVal = document.getElementById('prompt').value;
    const styleVal = document.getElementById('style').value;
    sceneTexts = splitIntoThreeScenes(promptVal);
    sceneImages = [];

    for (let i = 0; i < sceneTexts.length; i++) {
      statusEl.textContent = `Generating AI Scene ${i + 1} of ${sceneTexts.length}...`;
      barEl.style.width = `${15 + i * 20}%`;
      const seed = Math.floor(Math.random() * 999999);
      const img = await loadAiSceneImage(sceneTexts[i], styleVal, canvas.width, canvas.height, seed);
      sceneImages.push(img);
    }

    if (document.getElementById('voice').value === 'on' && window.AndroidBridge) {
      window.AndroidBridge.speakText(sceneTexts.join('. '));
    }
  }

  statusEl.textContent = 'Rendering video file...';
  const totalSeconds = Number(
    mode === 'ai' ? document.getElementById('duration').value : document.getElementById('imgDuration').value
  );

  const stream = canvas.captureStream(30);
  const mime = ['video/webm;codecs=vp9', 'video/webm;codecs=vp8', 'video/webm'].find(
    m => MediaRecorder.isTypeSupported(m)
  );

  if (!mime) {
    statusEl.textContent = 'WebM video recording is not supported on this device.';
    makeBtn.disabled = false;
    return;
  }

  lastMimeType = mime;
  const recorder = new MediaRecorder(stream, { mimeType: mime, videoBitsPerSecond: 4500000 });
  const chunks = [];

  recorder.ondataavailable = (e) => { if (e.data && e.data.size > 0) chunks.push(e.data); };
  recorder.onstop = () => {
    lastVideoBlob = new Blob(chunks, { type: mime });
    stream.getTracks().forEach(t => t.stop());
    barEl.style.width = '100%';
    statusEl.textContent = 'Done! Tap "Save Video to Phone Gallery" below.';
    saveBtn.classList.remove('hidden');
    makeBtn.disabled = false;
  };

  recorder.start();
  const startTime = performance.now();
  const motions = ['zoom', 'pan', 'out'];

  function renderLoop(now) {
    const elapsed = (now - startTime) / 1000;
    const t = Math.min(totalSeconds, elapsed);

    if (mode === 'ai') {
      const sceneLen = totalSeconds / sceneTexts.length;
      const idx = Math.min(sceneTexts.length - 1, Math.floor(t / sceneLen));
      const localT = t - idx * sceneLen;
      drawKenBurnsFrame(sceneImages[idx], localT, sceneLen, motions[idx % motions.length], sceneTexts[idx]);
    } else {
      drawKenBurnsFrame(
        uploadedImg,
        t,
        totalSeconds,
        document.getElementById('motion').value,
        document.getElementById('caption').value.trim()
      );
    }

    const renderPct = mode === 'ai' ? 75 + (t / totalSeconds) * 25 : (t / totalSeconds) * 100;
    barEl.style.width = `${Math.min(99, renderPct)}%`;

    if (elapsed < totalSeconds) {
      requestAnimationFrame(renderLoop);
    } else {
      recorder.stop();
    }
  }

  requestAnimationFrame(renderLoop);
}

function saveToGallery() {
  if (!lastVideoBlob) return;
  const statusEl = document.getElementById('status');
  statusEl.textContent = 'Saving to Movies/FreeMotion...';

  const reader = new FileReader();
  reader.onloadend = () => {
    const fileName = `FreeMotion_AI_${Date.now()}.webm`;
    if (window.AndroidBridge && window.AndroidBridge.saveBase64Video) {
      const result = window.AndroidBridge.saveBase64Video(reader.result, fileName, 'video/webm');
      statusEl.textContent = result === 'OK'
        ? `Saved to Movies/FreeMotion/${fileName}`
        : result;
    } else {
      // Browser fallback
      const a = document.createElement('a');
      a.href = URL.createObjectURL(lastVideoBlob);
      a.download = fileName;
      a.click();
    }
  };
  reader.readAsDataURL(lastVideoBlob);
}

updateCanvasSize();
</script>
</body>
</html>
          runOnUiThread {
                    Toast.makeText(
                        this@MainActivity,
                        "Saved to Movies/FreeMotion/$fileName",
                        Toast.LENGTH_LONG
                    ).show()
                }
                "OK"
            } catch (e: Exception) {
                "ERROR: ${e.message}"
            }
        }
    }
}

