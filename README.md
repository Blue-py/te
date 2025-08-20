<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>تفريغ فيديو وإظهار الكلمات — بدون خادم</title>
  <style>
    :root {
      --bg: #0f172a;           /* slate-900 */
      --card: #111827ee;       /* gray-900 */
      --muted: #94a3b8;        /* slate-400 */
      --text: #e5e7eb;         /* gray-200 */
      --accent: #22d3ee;       /* cyan-400 */
      --accent-2: #a78bfa;     /* violet-400 */
      --danger: #f43f5e;       /* rose-500 */
      --ok: #34d399;           /* emerald-400 */
    }
    *{box-sizing:border-box}
    body{
      margin:0; font-family: system-ui, -apple-system, Segoe UI, Roboto, "Noto Sans Arabic", Tahoma, Arial;
      background: radial-gradient(1200px 800px at 100% -10%, #1f2937, #0b1020), var(--bg);
      color:var(--text);
      min-height:100svh; display:flex; align-items:center; justify-content:center;
      padding:28px;
    }
    .app{width:min(1200px,100%);}
    header{display:flex; gap:16px; align-items:center; justify-content:space-between; margin-bottom:16px}
    h1{margin:0; font-size:clamp(1.2rem, 3vw, 1.6rem); letter-spacing:.3px}
    .hint{color:var(--muted); font-size:.95rem}
    .grid{display:grid; grid-template-columns: 1.1fr .9fr; gap:16px}
    .card{background:linear-gradient(180deg, #0b1324cc, #0b1324aa); border:1px solid #233054; border-radius:18px; box-shadow:0 8px 30px #0008; overflow:hidden}
    .card .hd{display:flex; align-items:center; justify-content:space-between; padding:14px 16px; border-bottom:1px solid #1e293b}
    .card .hd .title{font-weight:700; color:#e2e8f0}
    .body{padding:16px}

    .drop{
      border:2px dashed #334155; border-radius:14px; padding:18px; text-align:center; transition:.2s ease;
    }
    .drop.drag{border-color:var(--accent)}
    .drop input{display:none}
    .btns{display:flex; gap:8px; flex-wrap:wrap}
    .btn{
      border:1px solid #334155; background:#0b1324; color:#e2e8f0; padding:10px 14px; border-radius:12px; cursor:pointer; font-weight:600; transition:.15s;
    }
    .btn:hover{transform:translateY(-1px); border-color:#3b82f6}
    .btn.acc{background:linear-gradient(90deg, #0ea5e9 0%, #8b5cf6 100%); border:none; color:white}
    .btn.ok{background:linear-gradient(90deg, #10b981 0%, #22d3ee 100%); border:none; color:#001b2a}
    .btn.danger{background:#1f1320; border:1px solid #7f1d1d; color:#fecaca}

    video{width:100%; max-height:60svh; border-radius:12px; background:#000}

    .status{font-family:ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; font-size:.9rem; color:var(--muted)}

    .subs{
      min-height:180px; background: #0b1020; border:1px solid #1f2a44; border-radius:12px; padding:14px; line-height:1.9; font-size:1.15rem;
    }
    .word{opacity:.55; transition:opacity .15s ease, text-shadow .15s ease}
    .word.active{opacity:1; text-shadow:0 0 10px #22d3ee88, 0 0 20px #8b5cf666}
    .subs .line{display:inline}

    .progress{height:6px; background:#0b1324; border-radius:999px; overflow:hidden; border:1px solid #1e293b}
    .bar{height:100%; width:0%; background:linear-gradient(90deg, var(--accent), var(--accent-2)); transition:width .15s}

    footer{margin-top:14px; color:var(--muted); font-size:.86rem}
    code{background:#0b1324; border:1px solid #172038; padding:.2rem .45rem; border-radius:8px}
    .kbd{border:1px solid #263248; padding:.1rem .35rem; border-radius:6px; background:#0b1020; color:#cbd5e1}
  </style>
</head>
<body>
  <div class="app">
    <header>
      <h1>🎬 تفريغ فيديو تلقائي وإظهار الكلمات (نسخة متصفح فقط)</h1>
      <div class="hint">إرفع الفيديو → نولّد نصًا → كلمات تتوهّج حسب التوقيت
      </div>
    </header>

    <div class="grid">
      <!-- لوحة الفيديو -->
      <section class="card">
        <div class="hd">
          <div class="title">الفيديو</div>
          <div class="status" id="status">جاهز</div>
        </div>
        <div class="body">
          <label class="drop" id="drop">
            <input id="file" type="file" accept="video/MP4" />
            <div>اسحب الفيديو وأفلته هنا أو <span style="text-decoration:underline">إختر ملفًا</span></div>
            <div style="margin-top:8px; color:var(--muted); font-size:.9rem">يدعم MP4, WebM, MOV…</div>
          </label>
          <div style="height:12px"></div>
          <video id="video" controls playsinline></video>
          <div style="height:12px"></div>
          <div class="btns">
            <button class="btn acc" id="btnTranscribe">توليد النص (سريع/بدون موديل)</button>
            <button class="btn" id="btnVosk">توليد النص (دقّة أعلى مع Vosk)</button>
            <button class="btn danger" id="btnClear">مسح النص</button>
          </div>
          <div style="height:10px"></div>
          <div class="progress"><div class="bar" id="bar"></div></div>
        </div>
      </section>

      <!-- لوحة النص وكاريوكي الكلمات -->
      <section class="card">
        <div class="hd">
          <div class="title">الكلمات المتزامنة</div>
          <div class="btns">
            <label class="btn"><input id="srtFile" type="file" accept=".srt,.json" style="display:none">استيراد SRT/JSON</label>
            <button class="btn ok" id="btnExport">تصدير SRT</button>
          </div>
        </div>
        <div class="body">
          <div id="subs" class="subs" aria-live="polite"></div>
        </div>
      </section>
    </div>

    <footer>
      <p>🛈 يعمل زر <strong>"سريع/بدون موديل"</strong> باستخدام <strong>تعرف الكلام في المتصفح</strong> (بحاجة لسماح الميكروفون وتشغيل الفيديو بصوت مسموع). للمعالجة من الملف مباشرةً داخل المتصفح بدون إنترنت، استخدم زر <strong>Vosk</strong> بعد تنزيل ملفّات الموديل العربية ووضعها محليًا.</p>
      <p>
        إعداد Vosk (حل بدون خادم):
        <ol>
          <li>حمّل موديل العربية من مشروع <em>Vosk</em> (مثال: <code>vosk-model-small-ar-0.22</code>) وضعه في المسار <code>/models/ar/</code> بجانب هذا الملف.</li>
          <li>ضع ملفات المكتبة داخل <code>/libs/vosk/</code> (مثل <code>vosk.js</code> وملفات WebAssembly الخاصة به).</li>
          <li>ثم شغّل الموقع عبر <code>Live Server</code> أو أي خادم محلي (فتح الملف مباشرة قد يُقيّد الوصول لملفات WASM).</li>
        </ol>
      </p>
    </footer>
  </div>

  <!-- التطبيق -->
  <script type="module">
    // عناصر الواجهة
    const el = (id)=>document.getElementById(id);
    const fileInput = el('file');
    const drop = el('drop');
    const video = el('video');
    const statusEl = el('status');
    const bar = el('bar');
    const subsBox = el('subs');
    const btnTranscribe = el('btnTranscribe');
    const btnVosk = el('btnVosk');
    const btnClear = el('btnClear');
    const srtFile = el('srtFile');
    const btnExport = el('btnExport');

    let words = []; // {start,end,text}

    // واجهة جميلة للسحب والإفلات
    ;['dragenter','dragover'].forEach(evt=>drop.addEventListener(evt,e=>{e.preventDefault(); drop.classList.add('drag')}));
    ;['dragleave','drop'].forEach(evt=>drop.addEventListener(evt,e=>{e.preventDefault(); drop.classList.remove('drag')}));
    drop.addEventListener('drop', (e)=>{
      const f = e.dataTransfer.files?.[0];
      if (f) loadVideoFile(f);
    });
    drop.addEventListener('click', ()=> fileInput.click());
    fileInput.addEventListener('change', (e)=>{
      const f = e.target.files?.[0];
      if (f) loadVideoFile(f);
    });

    function setStatus(t){ statusEl.textContent = t }
    function setProgress(p){ bar.style.width = (p*100).toFixed(1)+'%' }

    function loadVideoFile(file){
      const url = URL.createObjectURL(file);
      video.src = url; video.load(); setStatus('تم تحميل الملف: '+file.name);
      words = []; renderWords();
    }

    // رندرة الكلمات مع تمييز الفعال حسب وقت الفيديو
    function renderWords(){
      subsBox.innerHTML = '';
      const frag = document.createDocumentFragment();
      for (const w of words){
        const span = document.createElement('span');
        span.className = 'word';
        span.dataset.start = w.start; span.dataset.end = w.end;
        span.textContent = w.text + ' ';
        frag.appendChild(span);
      }
      subsBox.appendChild(frag);
    }

    function highlightByTime(t){
      const spans = subsBox.querySelectorAll('.word');
      let lastActive;
      spans.forEach(s=>{
        const st = parseFloat(s.dataset.start), en = parseFloat(s.dataset.end);
        const active = t >= st && t <= en;
        s.classList.toggle('active', active);
        if (active) lastActive = s;
      });
      if (lastActive) lastActive.scrollIntoView({block:'center', inline:'nearest', behavior:'smooth'});
    }

    video.addEventListener('timeupdate', ()=> highlightByTime(video.currentTime));

    // استيراد SRT/JSON خارجي
    srtFile.addEventListener('change', async (e)=>{
      const f = e.target.files?.[0]; if (!f) return;
      const txt = await f.text();
      if (f.name.endsWith('.json')){
        const arr = JSON.parse(txt); // توقع: [{start,end,text}]
        words = arr; renderWords();
      } else {
        words = srtToWords(txt); renderWords();
      }
    });

    // تصدير SRT
    btnExport.addEventListener('click', ()=>{
      if (!words.length) return alert('لا يوجد كلمات للتصدير');
      const srt = wordsToSrt(words);
      const blob = new Blob([srt], {type:'text/plain;charset=utf-8'});
      const a = Object.assign(document.createElement('a'), {download:'captions.srt', href:URL.createObjectURL(blob)});
      a.click();
    });

    // زر مسح
    btnClear.addEventListener('click', ()=>{ words = []; renderWords(); setStatus('تم المسح'); setProgress(0) });

    // =====================
    // الطريقة 1: سريعة (تعرف المتصفح)
    // =====================
    // تعتمد على SpeechRecognition (تحتاج إذن ميكروفون وتشغيل الفيديو بصوت مسموع أمام الميك)
    btnTranscribe.addEventListener('click', async ()=>{
      const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
      if (!SR){
        alert('المتصفح لا يدعم SpeechRecognition. جرّب زر Vosk أو متصفح كروم/إيدج.');
        return;
      }
      if (!video.src) return alert('إرفع فيديو أولاً');
      setStatus('استماع… قم بتشغيل الفيديو بصوت مسموع');
      const rec = new SR();
      rec.lang = 'ar';
      rec.continuous = true;
      rec.interimResults = true;
      words = []; renderWords();
      let t0 = performance.now();

      rec.onresult = (e)=>{
        const now = (performance.now() - t0)/1000;
        let s = '';
        for (let i=e.resultIndex;i<e.results.length;i++){
          s = e.results[i][0].transcript.trim();
        }
        // نضيف الكلمات الجديدة كتقدير تقريبي في الزمن الحالي
        const ws = s.split(/\s+/).filter(Boolean);
        const dur = Math.max(0.25, ws.length*0.25);
        const start = Math.max(0, video.currentTime);
        const step = dur / Math.max(1, ws.length);
        ws.forEach((w, i)=>{
          words.push({start: start + i*step, end: start + (i+1)*step, text: w});
        });
        renderWords();
        setStatus('يلتقط الكلام…');
      }
      rec.onerror = (e)=>{ setStatus('خطأ التعرف: '+e.error) }
      rec.onend = ()=>{ setStatus('انتهى الاستماع') }
      rec.start();
    });

    // =====================
    // الطريقة 2: Vosk (بدون إنترنت ومن الملف مباشرة)
    // =====================
    // تتطلب تنزيل الموديل العربي ووضع ملفات المكتبة محليًا.
    btnVosk.addEventListener('click', async ()=>{
      if (!video.src) return alert('إرفع فيديو أولاً');
      try{
        setStatus('تحميل Vosk والموديل…'); setProgress(0.05);
        // نتوقع توفر كائن Vosk عالمي من /libs/vosk/vosk.js
        if (!window.Vosk) throw new Error('لم يتم العثور على مكتبة Vosk. ضعها في /libs/vosk/ واستوردها في هذا الملف.');

        // 1) تحويل صوت الفيديو إلى تدفق PCM 16kHz أحادي داخل المتصفح
        const stream = video.captureStream ? video.captureStream() : video.mozCaptureStream();
        if (!stream) throw new Error('المتصفح لا يدعم captureStream من الفيديو.');
        const audioTracks = stream.getAudioTracks();
        if (!audioTracks.length) throw new Error('لا يحتوي الفيديو على مسار صوتي.');

        const ac = new (window.AudioContext || window.webkitAudioContext)({sampleRate:16000});
        const src = ac.createMediaStreamSource(stream);
        const processor = ac.createScriptProcessor(4096, 1, 1);
        src.connect(processor); processor.connect(ac.destination);

        // 2) تهيئة Vosk
        const modelPath = '/models/ar'; // عدّل المسار حسب مكان الموديل
        const model = new Vosk.Model(modelPath);
        const rec = new Vosk.Recognizer({model, sampleRate: ac.sampleRate});

        words = []; renderWords();
        let lastPush = 0; setProgress(0.15);

        processor.onaudioprocess = (e)=>{
          const input = e.inputBuffer.getChannelData(0);
          // تحويل Float32 [-1..1] إلى Int16 PCM
          const pcm16 = new Int16Array(input.length);
          for (let i=0;i<input.length;i++){
            let s = Math.max(-1, Math.min(1, input[i]));
            pcm16[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
          }
          const now = ac.currentTime;
          if (now - lastPush > 0.05){ // دفعات
            rec.acceptWaveform(pcm16);
            lastPush = now;
            const part = rec.result();
            if (part && part.result){
              for (const w of part.result){ // {word,start,end}
                words.push({start:w.start, end:w.end, text:w.word});
              }
              renderWords();
            }
          }
        }

        video.play();
        setStatus('جاري التفريغ بواسطة Vosk… شغّل الفيديو');

        video.addEventListener('ended', ()=>{
          const final = rec.finalResult();
          if (final && final.result){
            for (const w of final.result){
              words.push({start:w.start, end:w.end, text:w.word});
            }
            renderWords();
          }
          rec.free(); model.free(); ac.close();
          setProgress(1); setStatus('تم التفريغ');
        }, {once:true});

      } catch(err){
        console.error(err);
        setStatus('تعذّر تشغيل Vosk: '+err.message);
      }
    });

    // ======= أدوات SRT بسيطة =======
    function pad2(n){return String(n).padStart(2,'0')}
    function fmtTime(t){
      const h = Math.floor(t/3600); t-=h*3600;
      const m = Math.floor(t/60); const s = Math.floor(t-m*60);
      const ms = Math.floor((t - Math.floor(t))*1000);
      return `${pad2(h)}:${pad2(m)}:${pad2(s)},${String(ms).padStart(3,'0')}`;
    }

    function wordsToSrt(ws){
      // نجمع كل 6-10 كلمات كسطر واحد
      const out=[]; let i=1; let buf=[];
      for (const w of ws){
        buf.push(w);
        if (buf.length>=8){
          const start=buf[0].start, end=buf[buf.length-1].end;
          out.push(`${i++}\n${fmtTime(start)} --> ${fmtTime(end)}\n${buf.map(x=>x.text).join(' ')}\n`);
          buf=[];
        }
      }
      if (buf.length){
        const start=buf[0].start, end=buf[buf.length-1].end;
        out.push(`${i++}\n${fmtTime(start)} --> ${fmtTime(end)}\n${buf.map(x=>x.text).join(' ')}\n`);
      }
      return out.join('\n');
    }

    function srtToWords(srt){
      const lines = srt.split(/\r?\n/);
      const ws=[]; let i=0;
      while(i<lines.length){
        // تخطّي رقم السطر
        if (/^\d+$/.test(lines[i])) i++;
        // سطر التوقيت
        const m = lines[i]?.match(/(\d\d):(\d\d):(\d\d),(\d\d\d)\s+-->\s+(\d\d):(\d\d):(\d\d),(\d\d\d)/);
        if (!m){i++; continue}
        const start = (+m[1])*3600 + (+m[2])*60 + (+m[3]) + (+m[4])/1000;
        const end   = (+m[5])*3600 + (+m[6])*60 + (+m[7]) + (+m[8])/1000;
        i++;
        let text='';
        while(i<lines.length && lines[i].trim()!==''){ text += lines[i]+' '; i++; }
        i++; // سطر فارغ
        const tokens = text.trim().split(/\s+/).filter(Boolean);
        const dur = Math.max(0.4, end-start);
        const step = dur/Math.max(1,tokens.length);
        tokens.forEach((w, k)=> ws.push({start:start+k*step, end:start+(k+1)*step, text:w}));
      }
      return ws;
    }
  </script>

  <!-- ملاحظة مهمة: لتفعيل Vosk، أضف في أسفل الصفحة (قبل إغلاق </body>):
  <script src="/libs/vosk/vosk.js"></script>
  ثم تأكد من وجود الموديل في /models/ar/
  -->
</body>
</html>
