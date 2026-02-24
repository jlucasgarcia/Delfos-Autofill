// ==UserScript==
// @name         Delfos Passenger Autofill
// @namespace    andiamo.delfos
// @version      1.2.0
// @description  Autofill datos de viajeros en Delfos
// @match        https://www.delfos.tur.ar/secure/*
// @match        https://delfos.tur.ar/secure/*
// @match        https://*.delfos.tur.ar/secure/*
// @grant        none
// @run-at       document-idle
// @updateURL    https://raw.githubusercontent.com/jlucasgarcia/Delfos-Autofill/main/delfos-autofill.user.js
// @downloadURL  https://raw.githubusercontent.com/jlucasgarcia/Delfos-Autofill/main/delfos-autofill.user.js
// ==/UserScript==

(function () {
  'use strict';

  const LS_KEY = 'delfos_autofill_json_v1';
  const LOG_PFX = '[DelfosFill v1.2]';

  const DOC_TYPE_MAP = {
    'DNI':           'IDENTITY_CARD',
    'IDENTITY_CARD': 'IDENTITY_CARD',
    'PASSPORT':      'PASSPORT',
    'PASAPORTE':     'PASSPORT',
    'NIE':           'NIE',
    'CUIT':          'IDENTITY_CARD',
    'CUIL':          'IDENTITY_CARD',
  };

  const COUNTRY_MAP = {
    'argentina': 'AR', 'ar': 'AR',
    'españa': 'ES', 'spain': 'ES', 'es': 'ES',
    'uruguay': 'UY', 'uy': 'UY',
    'brasil': 'BR', 'brazil': 'BR', 'br': 'BR',
    'chile': 'CL', 'cl': 'CL',
    'paraguay': 'PY', 'py': 'PY',
    'bolivia': 'BO', 'bo': 'BO',
    'peru': 'PE', 'pe': 'PE',
    'colombia': 'CO', 'co': 'CO',
    'mexico': 'MX', 'mx': 'MX',
    'estados unidos': 'US', 'usa': 'US', 'us': 'US',
  };

  /* =====================================================================
     UTILIDADES
  ===================================================================== */
  function log(...args) { console.log(LOG_PFX, ...args); }
  function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

  async function waitForAjax(timeoutMs = 2000) {
    const start = Date.now();
    while (Date.now() - start < timeoutMs) {
      try {
        const pf = window.PrimeFaces;
        if (pf && pf.ajax && pf.ajax.Queue) {
          if (pf.ajax.Queue.isEmpty()) break;
        } else {
          break;
        }
      } catch (_) { break; }
      await sleep(50);
    }
    await sleep(80);
  }

  async function typeIntoField(el, val) {
    if (!el || val == null) return false;
    const str = String(val).trim();
    if (!str) return false;

    el.focus();
    el.dispatchEvent(new Event('focus', { bubbles: true }));
    await sleep(20);
    el.value = '';

    for (const char of str) {
      el.dispatchEvent(new KeyboardEvent('keydown', { bubbles: true, cancelable: true, key: char }));
      el.value += char;
      el.dispatchEvent(new InputEvent('input', { bubbles: true, data: char, inputType: 'insertText' }));
      el.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true, cancelable: true, key: char }));
      await sleep(12);
    }

    el.dispatchEvent(new Event('change', { bubbles: true }));
    el.dispatchEvent(new Event('blur',   { bubbles: true }));
    log(`✓ typed [${el.name}] = "${el.value}"`);
    return el.value === str;
  }

  function setInput(el, val) {
    if (!el || val == null) return false;
    const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value')?.set;
    if (setter) setter.call(el, String(val));
    else el.value = String(val);
    ['input', 'change', 'blur'].forEach(ev => el.dispatchEvent(new Event(ev, { bubbles: true })));
    return true;
  }

  function setSelect(el, val) {
    if (!el || el.tagName !== 'SELECT') return false;
    const v = String(val).toLowerCase().trim();
    const opt =
      Array.from(el.options).find(o => o.value.toLowerCase() === v) ||
      Array.from(el.options).find(o => o.textContent.trim().toLowerCase() === v) ||
      Array.from(el.options).find(o => o.textContent.trim().toLowerCase().includes(v)) ||
      Array.from(el.options).find(o => o.value.toLowerCase().includes(v));
    if (!opt) { log(`⚠ SELECT sin match "${val}" en [${el.name}]`); return false; }
    el.value = opt.value;
    ['change', 'blur'].forEach(ev => el.dispatchEvent(new Event(ev, { bubbles: true })));
    log(`✓ SELECT [${el.name}] = "${opt.value}"`);
    return true;
  }

  function setSelectPicker(el, val) {
    if (!el || el.tagName !== 'SELECT') return false;
    const ok = setSelect(el, val);
    if (ok) { try { window.$(el).selectpicker('refresh'); } catch (_) {} }
    return ok;
  }

  /* =====================================================================
     HELPERS DE CAMPOS
  ===================================================================== */
  const byName = name => document.querySelector(`[name="${name}"]`);

  function findBirthInput(i) {
    const prefix = `passenger-form:distri:0:person:${i}:passengerData:`;
    return Array.from(document.querySelectorAll('input')).find(el =>
      el.name.startsWith(prefix) && el.name.endsWith(':birth_input')
    ) || null;
  }

  /* =====================================================================
     NORMALIZACIÓN
  ===================================================================== */
  function normalizeDate(s) {
    if (!s) return null;
    s = String(s).trim();
    if (/^\d{4}-\d{2}-\d{2}$/.test(s)) {
      const [y, m, d] = s.split('-');
      return `${d.padStart(2, '0')}/${m.padStart(2, '0')}/${y}`;
    }
    if (/^\d{1,2}\/\d{1,2}\/\d{4}$/.test(s)) {
      const [d, m, y] = s.split('/');
      return `${d.padStart(2, '0')}/${m.padStart(2, '0')}/${y}`;
    }
    return s;
  }

  function resolveDocType(val) {
    if (!val) return null;
    return DOC_TYPE_MAP[String(val).toUpperCase().trim()] || val;
  }

  function resolveCountry(val) {
    if (!val) return null;
    return COUNTRY_MAP[String(val).toLowerCase().trim()] || val;
  }

  function calcAge(birthDateStr) {
    if (!birthDateStr) return null;
    const s = String(birthDateStr).trim();
    let day, month, year;
    if (/^\d{4}-\d{2}-\d{2}$/.test(s)) {
      [year, month, day] = s.split('-').map(Number);
    } else if (/^\d{1,2}\/\d{1,2}\/\d{4}$/.test(s)) {
      [day, month, year] = s.split('/').map(Number);
    } else {
      return null;
    }
    const today = new Date();
    let age = today.getFullYear() - year;
    const m = today.getMonth() + 1 - month;
    if (m < 0 || (m === 0 && today.getDate() < day)) age--;
    return age;
  }

  function isAdult(p) {
    if (p.isAdult !== undefined) return Boolean(p.isAdult);
    if (p.type) {
      const t = String(p.type).toLowerCase();
      if (t === 'adult' || t === 'adulto') return true;
      if (t === 'child' || t === 'niño' || t === 'nino') return false;
    }
    const age = calcAge(p.birthDate);
    return age === null ? true : age >= 18;
  }

  /* =====================================================================
     LLENADO DE UN PASAJERO
  ===================================================================== */
  async function fillPassenger(p, i) {
    const base   = `passenger-form:distri:0:person:${i}:passengerData:`;
    const filled = [], failed = [], skipped = [];
    const adult  = isAdult(p);
    log(`Pax ${i + 1} — tipo: ${adult ? 'ADULTO' : 'NIÑO'} — edad: ${calcAge(p.birthDate)}`);

    async function track(key, fn) {
      try {
        const result = await Promise.resolve(fn());
        result ? filled.push(key) : failed.push(key);
      } catch (e) { failed.push(key); log(`✗ Error ${key}[${i}]:`, e); }
    }

    /* ---- documentType: setear sin AJAX primero ---- */
    const docTypeSel = byName(base + 'documentType');
    if (docTypeSel && p.documentType) {
      const resolvedType = resolveDocType(p.documentType);
      const opt =
        Array.from(docTypeSel.options).find(o => o.value.toLowerCase() === resolvedType.toLowerCase()) ||
        Array.from(docTypeSel.options).find(o => o.value.toLowerCase().includes(resolvedType.toLowerCase()));
      if (opt) {
        docTypeSel.value = opt.value;
        log(`✓ SELECT documentType seteado (sin AJAX) = "${opt.value}"`);
        filled.push('documentType');
      } else {
        log(`⚠ documentType sin match: "${resolvedType}"`);
        failed.push('documentType');
      }
    } else if (!p.documentType) {
      skipped.push('documentType');
    }

    /* ---- Campos de texto simples ---- */
    await track('nombre',    () => setInput(byName(base + 'nombre'),    p.nombre));
    await track('Apellidos', () => setInput(byName(base + 'Apellidos'), p.apellidos));
    await sleep(50);

    /* ---- documentNumber (caracter por caracter) ---- */
    if (p.documentNumber) {
      await track('documentNumber', async () => {
        const el = byName(base + 'documentNumber');
        if (!el) return false;
        const ok = await typeIntoField(el, String(p.documentNumber));
        if (!ok) {
          log(`⚠ Reintentando documentNumber[${i}]…`);
          await sleep(150);
          return await typeIntoField(el, String(p.documentNumber));
        }
        return ok;
      });
    } else {
      skipped.push('documentNumber');
    }

    /* ---- Disparar AJAX del documentType después de escribir documentNumber ---- */
    if (docTypeSel && p.documentType) {
      ['change', 'blur'].forEach(ev => docTypeSel.dispatchEvent(new Event(ev, { bubbles: true })));
      await waitForAjax(2000);
      log(`↻ AJAX post-documentType esperado`);

      /* Si el AJAX borró documentNumber, reescribir */
      if (p.documentNumber) {
        const el = byName(base + 'documentNumber');
        if (el && el.value !== String(p.documentNumber)) {
          log(`⚠ AJAX borró documentNumber[${i}], reescribiendo…`);
          const idx = filled.indexOf('documentNumber');
          if (idx > -1) filled.splice(idx, 1);
          const ok = await typeIntoField(el, String(p.documentNumber));
          ok ? filled.push('documentNumber') : failed.push('documentNumber');
        }
      }
    }

    /* ---- country (todos los pasajeros en Delfos) ---- */
    if (p.country) {
      await track('country', async () => {
        const el = byName(base + 'country');
        return el ? setSelectPicker(el, resolveCountry(p.country)) : false;
      });
      await sleep(150);
    } else {
      skipped.push('country');
    }

    /* ---- birthDate ---- */
    if (p.birthDate) {
      await track('birthDate', async () => {
        const el = findBirthInput(i);
        if (!el) return false;
        return await typeIntoField(el, normalizeDate(p.birthDate));
      });
    } else {
      skipped.push('birthDate');
    }

    /* ---- Email, teléfono y facturación solo Pax 0 ---- */
    if (i === 0) {
      if (p.email) await track('Email', () => setInput(byName(base + 'Email'), p.email));

      const phoneCountry = p.phoneCountry || p.country;
      if (phoneCountry) {
        await track('phoneCountry', async () => {
          const el = byName(base + 'phoneCountry');
          return el ? setSelectPicker(el, resolveCountry(phoneCountry)) : false;
        });
        await sleep(100);
      }

      if (p.phone) await track('phone', () => typeIntoField(byName(base + 'phone'), p.phone));

      const billingVal = p.billingnumber || p.cuit || p.cuil || p.documentNumber;
      if (billingVal) {
        await track('billingnumber', () => typeIntoField(byName(base + 'billingnumber'), String(billingVal)));
      } else {
        skipped.push('billingnumber');
      }
    }

    log(`Pax ${i + 1} → ✅[${filled}] ❌[${failed}] ⏭[${skipped}]`);
    return { passenger: i + 1, type: adult ? 'adulto' : 'niño', age: calcAge(p.birthDate), filled, failed, skipped };
  }

  /* =====================================================================
     ORDENAMIENTO: titular primero, adultos, niños menor→mayor
  ===================================================================== */
  function sortPassengers(passengers) {
    if (!passengers.length) return passengers;
    const [titular, ...rest] = passengers;
    const adults   = rest.filter(p =>  isAdult(p));
    const children = rest.filter(p => !isAdult(p));
    children.sort((a, b) => (calcAge(a.birthDate) ?? 0) - (calcAge(b.birthDate) ?? 0));
    const sorted = [titular, ...adults, ...children];
    log(`Orden final:`, sorted.map((p, i) =>
      `[${i}] ${p.nombre} ${p.apellidos} (${i === 0 ? 'TIT' : isAdult(p) ? 'adulto' : 'niño'} ${calcAge(p.birthDate)}a)`
    ));
    return sorted;
  }

  /* =====================================================================
     MOTOR PRINCIPAL
  ===================================================================== */
  async function runAutofill(txt, onProgress, onDone) {
    let data;
    try   { data = JSON.parse(txt); }
    catch (e) { onDone(null, `JSON inválido: ${e.message}`); return; }

    if (!Array.isArray(data.passengers)) {
      onDone(null, 'Falta la clave "passengers": [ ... ]');
      return;
    }

    const errors = data.passengers.flatMap((p, i) =>
      ['nombre', 'apellidos'].filter(k => !p[k]).map(k => `Pax ${i + 1}: falta "${k}"`)
    );
    if (errors.length) { onDone(null, errors.join('\n')); return; }

    data.passengers = sortPassengers(data.passengers);
    log(`Iniciando — ${data.passengers.length} pasajero(s)`);

    const summary = [];
    for (let i = 0; i < data.passengers.length; i++) {
      if (onProgress) onProgress(i, data.passengers.length);
      summary.push(await fillPassenger(data.passengers[i], i));
      await sleep(200);
    }

    onDone(summary, null);
  }

  /* =====================================================================
     UI
  ===================================================================== */
  function buildUI() {
    // Evitar doble inyección
    if (document.getElementById('df_fab')) return;

    injectStyles();

    const fab = document.createElement('button');
    fab.id = 'df_fab'; fab.type = 'button'; fab.textContent = '✈ Autofill';
    document.body.appendChild(fab);

    const panel = document.createElement('div');
    panel.id = 'df_panel';
    panel.innerHTML = `
      <div class="df_header">
        <span>✈ Delfos Autofill v1.2</span>
        <div style="display:flex;gap:6px">
          <button id="df_paste" class="df_btn df_orange">📋 Pegar</button>
          <button id="df_close" class="df_btn df_gray">✕</button>
        </div>
      </div>
      <div class="df_section">
        <small class="df_hint">
          <b>Campos:</b> <code>nombre</code>, <code>apellidos</code>,
          <code>documentType</code> (DNI/PASSPORT/NIE), <code>documentNumber</code>,
          <code>country</code>, <code>birthDate</code> (DD/MM/YYYY)<br>
          <b>Solo Pax 0:</b> <code>email</code>, <code>phone</code>,
          <code>phoneCountry</code>, <code>billingnumber</code><br>
          <b>Orden:</b> 1er JSON = titular · adultos · niños menor→mayor
        </small>
        <textarea id="df_json" placeholder='{"passengers":[{"nombre":"Juan","apellidos":"Garcia","documentType":"DNI","documentNumber":"20306619","country":"AR","birthDate":"15/04/1985","email":"j@email.com","phone":"1155443322"}]}'></textarea>
      </div>
      <div id="df_preview" class="df_section df_hidden"></div>
      <div class="df_section">
        <div style="display:flex;gap:6px;flex-wrap:wrap">
          <button id="df_fill"        class="df_btn df_green">▶ Autofill</button>
          <button id="df_preview_btn" class="df_btn df_orange">👁 Preview</button>
          <button id="df_save"        class="df_btn df_orange">💾 Guardar</button>
          <button id="df_clear"       class="df_btn df_red">🗑 Borrar</button>
        </div>
      </div>
      <div id="df_status" class="df_section df_hidden"></div>
      <div class="df_footer">
        <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>D</kbd> Autofill rápido &nbsp;|&nbsp;
        <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd> Pegar
      </div>`;
    document.body.appendChild(panel);

    fab.onclick = () => panel.classList.toggle('df_open');

    const area    = document.getElementById('df_json');
    const status  = document.getElementById('df_status');
    const preview = document.getElementById('df_preview');

    const stored = localStorage.getItem(LS_KEY);
    if (stored) area.value = stored;

    document.getElementById('df_close').onclick       = () => panel.classList.remove('df_open');
    document.getElementById('df_save').onclick        = () => { localStorage.setItem(LS_KEY, area.value.trim()); toast('💾 Guardado'); };
    document.getElementById('df_clear').onclick       = () => { area.value = ''; localStorage.removeItem(LS_KEY); clearStatus(); toast('🗑 Borrado'); };
    document.getElementById('df_preview_btn').onclick = () => showPreview(area.value.trim());

    document.getElementById('df_paste').onclick = async () => {
      try { area.value = await navigator.clipboard.readText(); toast('📋 Pegado'); }
      catch { toast('⚠ Permiso denegado — usá Ctrl+V', false); }
    };

    document.getElementById('df_fill').onclick = () => {
      const txt = area.value.trim() || localStorage.getItem(LS_KEY) || '';
      if (!txt) { toast('⚠ Ingresá un JSON primero', false); return; }
      startFill(txt);
    };

    document.addEventListener('keydown', e => {
      if (e.ctrlKey && e.shiftKey && e.key.toLowerCase() === 'd') {
        const txt = area.value.trim() || localStorage.getItem(LS_KEY) || '';
        txt ? startFill(txt) : toast('⚠ No hay JSON cargado', false);
      }
      if (e.ctrlKey && e.shiftKey && e.key.toLowerCase() === 'p') {
        document.getElementById('df_paste').click();
      }
    });

    function startFill(txt) {
      setStatus('⏳ Procesando…', 'info');
      runAutofill(
        txt,
        (i, total) => setStatus(`⏳ Llenando pasajero ${i + 1} de ${total}…`, 'info'),
        (summary, err) => {
          if (err) { setStatus(`❌ ${err}`, 'error'); toast(err, false); return; }
          const totalFilled = summary.reduce((a, s) => a + s.filled.length, 0);
          const totalFailed = summary.reduce((a, s) => a + s.failed.length, 0);
          setStatus(renderSummary(summary), 'result');
          toast(
            totalFailed ? `⚠ ${totalFilled} OK · ${totalFailed} fallaron` : `✅ ${totalFilled} campos completados`,
            totalFailed === 0
          );
        }
      );
    }

    function setStatus(html, type) {
      status.innerHTML = html;
      status.className = `df_section df_status_${type}`;
    }
    function clearStatus() { status.className = 'df_section df_hidden'; }

    function showPreview(txt) {
      if (!txt) { preview.classList.add('df_hidden'); return; }
      try {
        const data = JSON.parse(txt);
        if (!Array.isArray(data.passengers)) throw new Error('Falta "passengers"');
        preview.innerHTML = data.passengers.map((p, i) => {
          const age   = calcAge(p.birthDate);
          const adult = isAdult(p);
          const badge = i === 0
            ? `<span class="df_badge df_badge_tit">titular${age ? ` ${age}a` : ''}</span>`
            : adult
              ? `<span class="df_badge df_badge_adult">adulto${age ? ` ${age}a` : ''}</span>`
              : `<span class="df_badge df_badge_child">niño${age ? ` ${age}a` : ''}</span>`;
          return `
            <div class="df_pax">
              ${badge} <strong>${p.nombre || '?'} ${p.apellidos || '?'}</strong>
              ${p.documentNumber ? `<br><small>${p.documentType || 'DOC'}: ${p.documentNumber}</small>` : ''}
              ${p.birthDate ? `<small> · Nac: ${p.birthDate}</small>` : ''}
              ${p.email ? `<br><small>📧 ${p.email}</small>` : ''}
            </div>`;
        }).join('');
        preview.classList.remove('df_hidden');
      } catch (e) {
        preview.innerHTML = `<span class="df_err">❌ ${e.message}</span>`;
        preview.classList.remove('df_hidden');
      }
    }
  }

  /* =====================================================================
     RENDER SUMMARY
  ===================================================================== */
  function renderSummary(summary) {
    return summary.map(s => {
      const badge = s.passenger === 1
        ? `<span class="df_badge df_badge_tit">titular${s.age ? ` ${s.age}a` : ''}</span>`
        : s.type === 'adulto'
          ? `<span class="df_badge df_badge_adult">adulto${s.age ? ` ${s.age}a` : ''}</span>`
          : `<span class="df_badge df_badge_child">niño${s.age ? ` ${s.age}a` : ''}</span>`;
      return `
        <div class="df_pax">
          ${badge} <strong>Pax ${s.passenger}</strong>
          ${s.filled.length  ? `<br>✅ ${s.filled.join(', ')}` : ''}
          ${s.failed.length  ? `<br>❌ <b>${s.failed.join(', ')}</b>` : ''}
          ${s.skipped.length ? `<br>⏭ ${s.skipped.join(', ')}` : ''}
        </div>`;
    }).join('');
  }

  /* =====================================================================
     TOAST
  ===================================================================== */
  function toast(msg, ok = true) {
    let t = document.getElementById('df_toast');
    if (!t) { t = document.createElement('div'); t.id = 'df_toast'; document.body.appendChild(t); }
    t.className = ok ? 'df_toast_ok' : 'df_toast_err';
    t.textContent = msg;
    t.style.opacity = '1';
    clearTimeout(t._tm);
    t._tm = setTimeout(() => t.style.opacity = '0', 3200);
  }

  /* =====================================================================
     ESTILOS
  ===================================================================== */
  function injectStyles() {
    const css = `
      #df_fab {
        position: fixed !important; right: 20px !important; bottom: 20px !important;
        z-index: 2147483646 !important; padding: 11px 20px; background: #D97706;
        color: #fff; border: none; border-radius: 999px; cursor: pointer;
        font-size: 14px; font-weight: 700; box-shadow: 0 4px 20px rgba(0,0,0,.3);
        transition: transform .15s, background .15s; font-family: sans-serif;
      }
      #df_fab:hover { transform: scale(1.07); background: #B45309; }

      #df_panel {
        position: fixed !important; left: 20px !important; top: 80px !important;
        width: 390px; background: #fff; border: 1px solid #e5e7eb; border-radius: 12px;
        box-shadow: 0 16px 48px rgba(0,0,0,.18); z-index: 2147483645 !important;
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
        font-size: 12px; overflow: hidden; max-height: 0; opacity: 0;
        pointer-events: none; transition: max-height .3s ease, opacity .25s ease;
      }
      #df_panel.df_open { max-height: 92vh !important; opacity: 1 !important; pointer-events: all !important; overflow-y: auto; }

      .df_header {
        display: flex; align-items: center; justify-content: space-between;
        padding: 10px 12px; background: #D97706; color: #fff;
        font-size: 13px; font-weight: 700; position: sticky; top: 0; z-index: 1;
      }
      .df_section { padding: 8px 12px; border-top: 1px solid #f3f4f6; }
      .df_hidden  { display: none !important; }
      .df_hint    { color: #6b7280; line-height: 1.7; display: block; margin-bottom: 4px; }
      .df_hint code { background: #fef3c7; padding: 1px 4px; border-radius: 4px; font-size: 10px; }

      #df_json {
        width: 100%; height: 110px; box-sizing: border-box;
        border: 1px solid #d1d5db; border-radius: 6px; padding: 6px;
        font-family: monospace; font-size: 11px; resize: vertical; margin-top: 4px;
      }
      #df_json:focus { outline: 2px solid #D97706; border-color: transparent; }

      .df_btn { padding: 7px 10px; border: none; border-radius: 7px; cursor: pointer; font-size: 11px; font-weight: 600; transition: opacity .15s; }
      .df_btn:hover { opacity: .85; }
      .df_green  { background: #059669; color: #fff; }
      .df_orange { background: #D97706; color: #fff; }
      .df_red    { background: #DC2626; color: #fff; }
      .df_gray   { background: #e5e7eb; color: #374151; }

      .df_pax { padding: 5px 0; border-bottom: 1px solid #f3f4f6; line-height: 1.6; }
      .df_pax:last-child { border-bottom: none; }
      .df_err { color: #DC2626; }

      .df_badge {
        display: inline-block; padding: 1px 6px; border-radius: 999px;
        font-size: 10px; font-weight: 700; margin-right: 4px;
      }
      .df_badge_tit   { background: #fef3c7; color: #92400e; }
      .df_badge_adult { background: #dbeafe; color: #1d4ed8; }
      .df_badge_child { background: #d1fae5; color: #065f46; }

      .df_status_info   { background: #fffbeb; }
      .df_status_error  { background: #fef2f2; color: #991b1b; }
      .df_status_result { background: #f0fdf4; }

      .df_footer {
        padding: 6px 12px; background: #f9fafb; color: #9ca3af;
        font-size: 11px; border-top: 1px solid #e5e7eb;
      }
      .df_footer kbd { background: #e5e7eb; border-radius: 3px; padding: 1px 4px; font-size: 10px; font-family: monospace; }

      #df_toast {
        position: fixed !important; left: 16px !important; bottom: 20px !important;
        z-index: 2147483647 !important; padding: 10px 16px; border-radius: 8px;
        font-size: 12px; font-weight: 500; box-shadow: 0 6px 20px rgba(0,0,0,.2);
        opacity: 0; transition: opacity .25s; max-width: 320px;
      }
      .df_toast_ok  { background: #111827; color: #fff; }
      .df_toast_err { background: #991B1B; color: #fff; }
    `;
    const s = document.createElement('style');
    s.textContent = css;
    document.head.appendChild(s);
  }

  /* =====================================================================
     INIT — mismo patrón robusto que Mitika
  ===================================================================== */
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', buildUI);
  } else {
    buildUI();
  }

})();
