# Painel de Geração de Anúncios (anuncioinsta) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Flask panel that generates Instagram ad creatives (2 images,
1 video, copy variations) for pulse.store26 from uploaded product photos, using the
Gemini API.

**Architecture:** Flask app with two routes (`GET /`, `POST /gerar`), a
`gemini_client.py` module wrapping three separate Gemini model calls (image
generation, video generation, text generation — see Global Constraints for why three
models instead of one), and a single-page frontend that uploads photos via
`FormData`/`fetch`, shows a spinner, then renders results as Instagram phone mockups
reusing the visual style already established in `anuncioinsta/previa_anuncio.html`.

**Tech Stack:** Python 3, Flask, `google-genai` SDK, `python-dotenv`. No frontend
framework — plain HTML/CSS/JS in `templates/index.html`.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-02-anuncioinsta-gemini-design.md`.
- **Model split (confirmed against official docs during brainstorming — see spec
  session):** `gemini-omni-flash-preview` outputs **video only**, it cannot generate
  static images or text. Use `gemini-3.1-flash-image` for the two static images and
  `gemini-3.5-flash` for copy text. All three are called from `gemini_client.py`.
- `google-genai` client (`from google import genai; client = genai.Client()`) reads
  the API key from the `GEMINI_API_KEY` environment variable automatically — no need
  to pass `api_key=` explicitly, as long as `python-dotenv` has loaded `.env` into
  `os.environ` before `genai.Client()` is constructed.
- Output aspect ratio for image calls goes in
  `response_format={"type": "image", "aspect_ratio": "<ratio>"}`; for video calls,
  `response_format={"type": "video", "aspect_ratio": "<ratio>"}`. One image is
  returned per call (`interaction.output_image.data`, base64) — call the image model
  twice for two aspect ratios (`4:5` and `1:1`). Video is
  `interaction.output_video.data` (base64), aspect ratio `9:16`. Text is
  `interaction.output_text`.
- Port: **8759**.
- All new files live under `C:\Projetos\APLICATIVO GABRIEL\anuncioinsta\`. Do not
  move, rename, or delete the existing loose photos/videos/`previa_anuncio.html`
  already in that folder.
- Testing approach per spec: **manual verification only**, no automated test suite
  (personal internal tool, matches the pattern of the user's other panels). Every
  task below ends with a manual run-and-check step instead of a pytest cycle.
- `anuncioinsta/.env` holds `GEMINI_API_KEY`. The repo root `.gitignore` already has
  `.env` and `*.env` (unscoped, so it covers this subfolder too) — do not add a
  redundant `.gitignore` inside `anuncioinsta/`.

---

### Task 1: Project scaffolding

**Files:**
- Create: `anuncioinsta/requirements.txt`
- Create: `anuncioinsta/.env`
- Create: `anuncioinsta/campanhas/.gitkeep`

**Interfaces:**
- Produces: a `campanhas/` directory that Task 3 writes generated output into, and an
  installed environment (`flask`, `google-genai`, `python-dotenv`) that Tasks 2–3
  import.

- [ ] **Step 1: Create `requirements.txt`**

```
flask>=3.1.0
google-genai>=1.5.0
python-dotenv>=1.0.1
```

Note: `google-genai` is a fast-moving SDK (the `client.interactions.create()` API
used below is current as of mid-2026). Pinning to an exact old version risks
installing a release that predates this API — use `>=` floors so `pip install`
always grabs the latest compatible release.

- [ ] **Step 2: Create `.env` with the real API key**

Create `anuncioinsta/.env`:

```
GEMINI_API_KEY=AIzaSyDYmiKVxQT55O2VCrt6vtSEE2evKPPejQs
```

- [ ] **Step 3: Create the campanhas output directory**

Create `anuncioinsta/campanhas/.gitkeep` (empty file) so the directory exists even
before any generation has run.

- [ ] **Step 4: Install dependencies and verify**

Run (from `C:\Projetos\APLICATIVO GABRIEL\anuncioinsta`):
```bash
pip install -r requirements.txt
```
Expected: all three packages install without error.

Run:
```bash
python -c "from google import genai; import flask, dotenv; print('ok')"
```
Expected output: `ok`

- [ ] **Step 5: Commit**

```bash
git add anuncioinsta/requirements.txt anuncioinsta/campanhas/.gitkeep
git commit -m "chore: scaffold anuncioinsta panel dependencies and output dir"
```

Note: `anuncioinsta/.env` is intentionally NOT committed (matches root `.gitignore`
pattern `*.env`). Verify with `git status` that it does not appear as staged/tracked.

---

### Task 2: `gemini_client.py` — image, video and copy generation

**Files:**
- Create: `anuncioinsta/gemini_client.py`

**Interfaces:**
- Consumes: `GEMINI_API_KEY` from environment (loaded by whoever imports this module
  before calling its functions — Task 3's `app.py` calls `load_dotenv()` first).
- Produces (used by Task 3):
  - `generate_image(photos: list[bytes], mime_types: list[str], prompt: str, aspect_ratio: str) -> bytes`
  - `generate_video(photo: bytes, mime_type: str, prompt: str, aspect_ratio: str = "9:16") -> bytes`
  - `generate_copy(marca: str, promocao: str, loja: str, cta: str) -> dict` with keys
    `textos_principais` (list[str]), `headlines` (list[str]), `descricoes` (list[str])

- [ ] **Step 1: Write `gemini_client.py`**

```python
import base64
import json

from google import genai

IMAGE_MODEL = "gemini-3.1-flash-image"
VIDEO_MODEL = "gemini-omni-flash-preview"
TEXT_MODEL = "gemini-3.5-flash"


def _client():
    return genai.Client()


def generate_image(photos, mime_types, prompt, aspect_ratio):
    client = _client()
    input_blocks = [{"type": "text", "text": prompt}]
    for data, mime in zip(photos, mime_types):
        input_blocks.append({
            "type": "image",
            "data": base64.b64encode(data).decode("utf-8"),
            "mime_type": mime,
        })
    interaction = client.interactions.create(
        model=IMAGE_MODEL,
        input=input_blocks,
        response_format={"type": "image", "aspect_ratio": aspect_ratio},
    )
    return base64.b64decode(interaction.output_image.data)


def generate_video(photo, mime_type, prompt, aspect_ratio="9:16"):
    client = _client()
    interaction = client.interactions.create(
        model=VIDEO_MODEL,
        input=[
            {
                "type": "image",
                "data": base64.b64encode(photo).decode("utf-8"),
                "mime_type": mime_type,
            },
            {"type": "text", "text": prompt},
        ],
        response_format={"type": "video", "aspect_ratio": aspect_ratio},
    )
    return base64.b64decode(interaction.output_video.data)


def generate_copy(marca, promocao, loja, cta):
    client = _client()
    prompt = (
        "Você é um redator publicitário brasileiro especializado em anúncios do "
        "Instagram (Meta Ads). "
        f"Loja: {loja}. Marca(s) em destaque: {marca}. "
        f"Promoção/diferencial: {promocao or 'nenhuma'}. "
        f'Botão de call-to-action fixo: "{cta}". '
        "Gere variações de copy para um anúncio patrocinado. "
        "Responda SOMENTE com um JSON válido, sem markdown e sem texto fora do "
        "JSON, no formato exato: "
        '{"textos_principais": ["...", "...", "..."], '
        '"headlines": ["...", "...", "..."], '
        '"descricoes": ["...", "..."]}. '
        "textos_principais: até 125 caracteres cada, 3 variações. "
        "headlines: até 40 caracteres cada, 3 variações. "
        "descricoes: até 30 caracteres cada, 2 variações. "
        "Tom direto, comercial, adequado para vendas via direct do Instagram."
    )
    interaction = client.interactions.create(model=TEXT_MODEL, input=prompt)
    text = interaction.output_text.strip()
    if text.startswith("```"):
        text = text.strip("`")
        if text.startswith("json"):
            text = text[4:]
        text = text.strip()
    return json.loads(text)
```

- [ ] **Step 2: Manually verify `generate_copy` against the real API**

Run:
```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python -c "
from dotenv import load_dotenv
load_dotenv()
import gemini_client
copy = gemini_client.generate_copy('Nike, Mizuno', 'Frete grátis', 'pulse.store26 · Alvorada/RS', 'Enviar Mensagem')
print(copy)
assert 'textos_principais' in copy and 'headlines' in copy and 'descricoes' in copy
print('OK')
"
```
Expected: prints a dict with the three keys, then `OK`. If it raises
`json.JSONDecodeError`, print `text` before the `json.loads` call to see the raw
model output and adjust the prompt/stripping logic accordingly — do not move on
until this passes with real output.

- [ ] **Step 3: Manually verify `generate_image` against the real API**

Use one real product photo from the existing folder as input:
```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python -c "
from dotenv import load_dotenv
load_dotenv()
import gemini_client
with open('cv_1_nike.jpeg', 'rb') as f:
    photo = f.read()
img = gemini_client.generate_image([photo], ['image/jpeg'], 'Anúncio de Instagram para boné Nike, fundo de estúdio limpo, estilo comercial.', '4:5')
with open('_test_output_image.png', 'wb') as f:
    f.write(img)
print(len(img), 'bytes written')
"
```
Expected: prints a byte count > 0. Open `_test_output_image.png` and confirm it's a
plausible ad image. Delete `_test_output_image.png` afterward (it's a throwaway
manual test artifact, not part of the app).

- [ ] **Step 4: Manually verify `generate_video` against the real API**

```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python -c "
from dotenv import load_dotenv
load_dotenv()
import gemini_client
with open('cv_1_nike.jpeg', 'rb') as f:
    photo = f.read()
video = gemini_client.generate_video(photo, 'image/jpeg', 'Vídeo curto de anúncio para Instagram Story, boné Nike em destaque, movimento suave de câmera.')
with open('_test_output_video.mp4', 'wb') as f:
    f.write(video)
print(len(video), 'bytes written')
"
```
Expected: prints a byte count > 0. Play `_test_output_video.mp4` and confirm it's a
plausible short vertical ad clip. Delete `_test_output_video.mp4` afterward.

- [ ] **Step 5: Commit**

```bash
git add anuncioinsta/gemini_client.py
git commit -m "feat: add gemini_client wrapper for image, video and copy generation"
```

---

### Task 3: `app.py` — Flask routes

**Files:**
- Create: `anuncioinsta/app.py`

**Interfaces:**
- Consumes: `gemini_client.generate_image`, `gemini_client.generate_video`,
  `gemini_client.generate_copy` (Task 2).
- Produces (used by Task 4's frontend):
  - `GET /` → renders `templates/index.html`
  - `POST /gerar` (multipart form: `fotos` file list, `marca`, `promocao`, `loja`,
    `cta` text fields) → JSON
    `{"imagens": [str], "video": str|null, "copy": dict|null, "erros": [str], "pasta": str}`
  - `GET /campanhas/<pasta>/<arquivo>` → serves a generated file for
    download/preview

- [ ] **Step 1: Write `app.py`**

```python
import json
import re
import unicodedata
from datetime import date
from pathlib import Path

from dotenv import load_dotenv

load_dotenv()

from flask import Flask, jsonify, render_template, request, send_from_directory

import gemini_client

app = Flask(__name__)

BASE_DIR = Path(__file__).parent
CAMPANHAS_DIR = BASE_DIR / "campanhas"

LOJA_PADRAO = "pulse.store26 · Alvorada/RS"
CTA_PADRAO = "Enviar Mensagem"


def slugify(texto):
    texto = unicodedata.normalize("NFKD", texto).encode("ascii", "ignore").decode("ascii")
    texto = re.sub(r"[^a-zA-Z0-9]+", "-", texto).strip("-").lower()
    return texto or "campanha"


@app.route("/")
def index():
    return render_template("index.html", loja_padrao=LOJA_PADRAO, cta_padrao=CTA_PADRAO)


@app.route("/gerar", methods=["POST"])
def gerar():
    fotos = request.files.getlist("fotos")
    marca = request.form.get("marca", "").strip()
    promocao = request.form.get("promocao", "").strip()
    loja = request.form.get("loja", "").strip() or LOJA_PADRAO
    cta = request.form.get("cta", "").strip() or CTA_PADRAO

    if not fotos or fotos[0].filename == "":
        return jsonify({"erro": "Envie ao menos uma foto do produto."}), 400
    if not marca:
        return jsonify({"erro": "Informe a marca."}), 400

    fotos_bytes = [f.read() for f in fotos]
    fotos_mime = [f.mimetype for f in fotos]

    pasta = CAMPANHAS_DIR / f"{date.today().isoformat()}_{slugify(marca)}"
    pasta.mkdir(parents=True, exist_ok=True)

    resultado = {"imagens": [], "video": None, "copy": None, "erros": [], "pasta": pasta.name}

    prompt_base = (
        f"Crie um anúncio de Instagram profissional para a loja {loja}, "
        f"vendendo produtos da marca {marca}. {promocao or ''} "
        "Use como referência fiel as fotos do produto anexadas — mantenha o "
        "produto real, não invente outro modelo. Fundo limpo, iluminação de "
        "estúdio, estilo comercial para Meta Ads."
    )

    for nome_formato, aspect in (("feed_1", "4:5"), ("feed_2", "1:1")):
        try:
            img_bytes = gemini_client.generate_image(fotos_bytes, fotos_mime, prompt_base, aspect)
            caminho = pasta / f"img_{nome_formato}.png"
            caminho.write_bytes(img_bytes)
            resultado["imagens"].append(caminho.name)
        except Exception as exc:
            resultado["erros"].append(f"Falha ao gerar imagem ({aspect}): {exc}")

    try:
        prompt_video = prompt_base + " Vídeo curto no formato Story, produto em destaque, movimento suave."
        video_bytes = gemini_client.generate_video(fotos_bytes[0], fotos_mime[0], prompt_video)
        caminho_video = pasta / "video_story.mp4"
        caminho_video.write_bytes(video_bytes)
        resultado["video"] = caminho_video.name
    except Exception as exc:
        resultado["erros"].append(f"Falha ao gerar vídeo: {exc}")

    try:
        copy = gemini_client.generate_copy(marca, promocao, loja, cta)
        (pasta / "copy.json").write_text(
            json.dumps(copy, ensure_ascii=False, indent=2), encoding="utf-8"
        )
        resultado["copy"] = copy
    except Exception as exc:
        resultado["erros"].append(f"Falha ao gerar copy: {exc}")

    return jsonify(resultado)


@app.route("/campanhas/<pasta>/<arquivo>")
def servir_arquivo(pasta, arquivo):
    return send_from_directory(CAMPANHAS_DIR / pasta, arquivo)


if __name__ == "__main__":
    app.run(port=8759, debug=True)
```

- [ ] **Step 2: Manually verify `slugify` and validation logic**

Run:
```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python -c "
from app import slugify
print(slugify('Nike, Mizuno'))
print(slugify('Quiksilver & Lacoste'))
"
```
Expected output:
```
nike-mizuno
quiksilver-lacoste
```

- [ ] **Step 3: Manually verify validation errors without calling the Gemini API**

Start the server:
```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python app.py
```
In a second terminal:
```bash
curl -s -X POST http://localhost:8759/gerar -F "marca=Nike"
```
Expected: HTTP 400 with `{"erro": "Envie ao menos uma foto do produto."}` (no photo
sent — this must fail before any Gemini API call happens, so it should return
instantly).

```bash
curl -s -X POST http://localhost:8759/gerar -F "fotos=@cv_1_nike.jpeg"
```
Expected: HTTP 400 with `{"erro": "Informe a marca."}`.

Stop the server (Ctrl+C) once both checks pass. Task 5 does the full real-generation
end-to-end check once the frontend exists.

- [ ] **Step 4: Commit**

```bash
git add anuncioinsta/app.py
git commit -m "feat: add Flask app with / and /gerar routes for anuncioinsta panel"
```

---

### Task 4: `templates/index.html` — panel UI

**Files:**
- Create: `anuncioinsta/templates/index.html`

**Interfaces:**
- Consumes: `POST /gerar` (Task 3), file `GET /campanhas/<pasta>/<arquivo>` (Task 3),
  template variables `loja_padrao` and `cta_padrao` (Task 3's `index()` route).

- [ ] **Step 1: Write `templates/index.html`**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Gerar Anúncio — pulse.store26</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #0f0f1a;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    color: #fff;
    padding: 40px 24px 80px;
  }
  h1 { font-size: 22px; margin-bottom: 24px; text-align: center; }
  form {
    max-width: 480px;
    margin: 0 auto 40px;
    background: rgba(255,255,255,.04);
    border: 1px solid rgba(255,255,255,.09);
    border-radius: 14px;
    padding: 24px;
  }
  label { display: block; font-size: 12px; color: #999; margin: 14px 0 6px; text-transform: uppercase; letter-spacing: 1px; }
  input[type="text"], input[type="file"] {
    width: 100%; padding: 10px 12px; border-radius: 8px;
    border: 1px solid rgba(255,255,255,.15); background: #1a1a2a; color: #fff; font-size: 14px;
  }
  button {
    margin-top: 20px; width: 100%; padding: 12px; border: none; border-radius: 8px;
    background: #0095f6; color: #fff; font-weight: 700; font-size: 14px; cursor: pointer;
  }
  button:disabled { opacity: .5; cursor: not-allowed; }
  #status { text-align: center; margin: 16px 0; color: #999; font-size: 13px; }
  #erros { max-width: 480px; margin: 0 auto 20px; color: #f5a623; font-size: 13px; }
  #erros p { margin-bottom: 6px; }
  .resultado { max-width: 960px; margin: 0 auto; display: flex; gap: 24px; flex-wrap: wrap; justify-content: center; }
  .card { background: rgba(255,255,255,.04); border: 1px solid rgba(255,255,255,.09); border-radius: 14px; padding: 16px; width: 260px; text-align: center; }
  .card img, .card video { width: 100%; border-radius: 8px; margin-bottom: 10px; }
  .card a { color: #5ab8ff; font-size: 12px; text-decoration: none; }
  .copy-block { max-width: 960px; margin: 32px auto 0; }
  .copy-block h2 { font-size: 16px; margin-bottom: 12px; text-align: center; }
  .copy-card { background: rgba(255,255,255,.04); border: 1px solid rgba(255,255,255,.09); border-radius: 10px; padding: 14px 16px; margin-bottom: 10px; font-size: 13px; }
  .copy-card b { color: #5ab8ff; display: block; margin-bottom: 6px; font-size: 11px; text-transform: uppercase; letter-spacing: 1px; }
</style>
</head>
<body>

<h1>Gerar Anúncio — pulse.store26</h1>

<form id="form-gerar">
  <label for="fotos">Fotos do produto</label>
  <input type="file" id="fotos" name="fotos" accept="image/*" multiple required>

  <label for="marca">Marca(s)</label>
  <input type="text" id="marca" name="marca" placeholder="Ex: Nike, Mizuno" required>

  <label for="promocao">Promoção / diferencial</label>
  <input type="text" id="promocao" name="promocao" placeholder="Ex: Frete grátis">

  <label for="loja">Loja / cidade</label>
  <input type="text" id="loja" name="loja" value="{{ loja_padrao }}">

  <label for="cta">CTA</label>
  <input type="text" id="cta" name="cta" value="{{ cta_padrao }}">

  <button type="submit" id="btn-gerar">Gerar</button>
</form>

<div id="status"></div>
<div id="erros"></div>
<div class="resultado" id="resultado"></div>
<div class="copy-block" id="copy-block"></div>

<script>
const form = document.getElementById('form-gerar');
const btn = document.getElementById('btn-gerar');
const status = document.getElementById('status');
const errosDiv = document.getElementById('erros');
const resultadoDiv = document.getElementById('resultado');
const copyBlock = document.getElementById('copy-block');

form.addEventListener('submit', async (e) => {
  e.preventDefault();
  btn.disabled = true;
  status.textContent = 'Gerando criativos, isso pode levar um minuto...';
  errosDiv.innerHTML = '';
  resultadoDiv.innerHTML = '';
  copyBlock.innerHTML = '';

  const formData = new FormData(form);
  try {
    const resp = await fetch('/gerar', { method: 'POST', body: formData });
    const data = await resp.json();

    if (!resp.ok) {
      errosDiv.innerHTML = `<p>${data.erro}</p>`;
      status.textContent = '';
      btn.disabled = false;
      return;
    }

    status.textContent = 'Concluído.';
    if (data.erros && data.erros.length) {
      errosDiv.innerHTML = data.erros.map(e => `<p>${e}</p>`).join('');
    }

    (data.imagens || []).forEach(nome => {
      const url = `/campanhas/${data.pasta}/${nome}`;
      resultadoDiv.innerHTML += `
        <div class="card">
          <img src="${url}" alt="${nome}">
          <a href="${url}" download>Baixar imagem</a>
        </div>`;
    });

    if (data.video) {
      const url = `/campanhas/${data.pasta}/${data.video}`;
      resultadoDiv.innerHTML += `
        <div class="card">
          <video src="${url}" controls></video>
          <a href="${url}" download>Baixar vídeo</a>
        </div>`;
    }

    if (data.copy) {
      let html = '<h2>Variações de Copy</h2>';
      html += '<div class="copy-card"><b>Textos Principais</b>' + data.copy.textos_principais.map(t => `<p>${t}</p>`).join('') + '</div>';
      html += '<div class="copy-card"><b>Headlines</b>' + data.copy.headlines.map(t => `<p>${t}</p>`).join('') + '</div>';
      html += '<div class="copy-card"><b>Descrições</b>' + data.copy.descricoes.map(t => `<p>${t}</p>`).join('') + '</div>';
      copyBlock.innerHTML = html;
    }
  } catch (err) {
    errosDiv.innerHTML = `<p>Erro de conexão: ${err.message}</p>`;
    status.textContent = '';
  } finally {
    btn.disabled = false;
  }
});
</script>

</body>
</html>
```

- [ ] **Step 2: Manually verify the form renders**

```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python app.py
```
Open `http://localhost:8759` in a browser. Expected: the form appears with "Loja /
cidade" pre-filled as `pulse.store26 · Alvorada/RS` and "CTA" pre-filled as `Enviar
Mensagem`. Stop the server (Ctrl+C).

- [ ] **Step 3: Commit**

```bash
git add anuncioinsta/templates/index.html
git commit -m "feat: add anuncioinsta panel frontend form and results view"
```

---

### Task 5: End-to-end manual verification

**Files:** none (verification only).

- [ ] **Step 1: Run a full real generation through the browser**

```bash
cd "C:/Projetos/APLICATIVO GABRIEL/anuncioinsta"
python app.py
```
Open `http://localhost:8759`. Upload 2-3 real product photos from the folder (e.g.
`cv_1_nike.jpeg`, `cv_2_nike.jpeg`), fill Marca = `Nike`, Promoção = `Frete grátis`,
leave Loja and CTA as default, click **Gerar**.

Expected:
- Spinner text appears, then within roughly a minute the two images and one video
  render on the page with working "Baixar" links.
- The copy section shows 3 textos principais, 3 headlines, 2 descrições.
- A new folder `anuncioinsta/campanhas/2026-07-02_nike/` exists containing
  `img_feed_1.png`, `img_feed_2.png`, `video_story.mp4`, `copy.json` — open each file
  directly to confirm they are valid, non-empty, and visually plausible ad assets for
  Nike caps.

If any single piece fails (e.g. video generation errors out), confirm the error
message appears in the page's error area while the successful pieces (images/copy)
still render — this is the partial-failure behavior from the spec.

- [ ] **Step 2: Confirm the API key was never committed**

```bash
cd "C:/Projetos/APLICATIVO GABRIEL"
git log --all -p -- anuncioinsta/.env
git status --short anuncioinsta/
```
Expected: the `git log` command shows no output (the file was never committed), and
`git status` does not list `anuncioinsta/.env` as tracked/staged (it may appear as an
untracked/ignored file, which is correct).

- [ ] **Step 3: Stop the server**

Ctrl+C in the terminal running `python app.py`.

No commit for this task — it's verification only, no files change.
