<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Child Vaccination Monitoring — Prototype UI</title>
  <style>
    :root{--bg:#f6f8fb;--card:#fff;--muted:#6b7280;--accent:#2563eb}
    body{font-family:Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial; background:var(--bg); margin:0; padding:24px}
    .container{max-width:1000px; margin:0 auto}
    header{display:flex;align-items:center;gap:16px;margin-bottom:20px}
    h1{margin:0;font-size:20px}
    .grid{display:grid;grid-template-columns:1fr 360px;gap:18px}
    .card{background:var(--card);border-radius:10px;padding:16px;box-shadow:0 6px 18px rgba(15,23,42,0.06)}
    label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
    input,select,button{width:100%;padding:10px;border-radius:8px;border:1px solid #e6e9ef;font-size:14px}
    button{background:var(--accent);color:#fff;border:none;cursor:pointer}
    .muted{color:var(--muted);font-size:13px}
    .list{display:flex;flex-direction:column;gap:12px}
    .child-row{display:flex;justify-content:space-between;align-items:center;padding:10px;border-radius:8px;border:1px solid #f1f3f6}
    .dose{padding:8px;border-radius:8px;border:1px dashed #e6e9ef;display:flex;justify-content:space-between;align-items:center}
    .small{font-size:13px}
    .success{background:#ecfdf5;color:#065f46;padding:6px;border-radius:6px}
    .danger{background:#fff1f2;color:#9f1239;padding:6px;border-radius:6px}
    .help{font-size:12px;color:#374151}
    footer{margin-top:18px;color:var(--muted);font-size:13px}
    @media (max-width:880px){.grid{grid-template-columns:1fr}}
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>Child Vaccination Monitoring — Prototype UI</h1>
      <div class="muted">Frontend demo (static HTML + JS)</div>
    </header>

    <div class="grid">
      <section class="card">
        <h3 style="margin-top:0">Create Guardian</h3>
        <div class="muted small">Create a guardian (parent/healthworker). The backend should expose POST /api/users — if not, create users directly in DB for the prototype.</div>
        <div style="margin-top:12px">
          <label for="g-name">Name</label>
          <input id="g-name" placeholder="Guardian name" />
          <label for="g-phone" style="margin-top:8px">Phone (+countrycode)</label>
          <input id="g-phone" placeholder="+919876543210" />
          <label for="g-role" style="margin-top:8px">Role</label>
          <select id="g-role"><option value="parent">Parent</option><option value="healthworker">Healthworker</option></select>
          <div style="margin-top:12px"><button id="btn-create-guardian">Create Guardian</button></div>
          <div id="create-guardian-result" class="help" style="margin-top:8px"></div>
        </div>

        <hr style="margin:16px 0" />

        <h3 style="margin-top:0">Add Child</h3>
        <div class="muted small">Add a child; this will call POST /api/children and auto-generate schedules on the backend (see prototype server).</div>
        <div style="margin-top:12px">
          <label for="child-guardian">Guardian</label>
          <select id="child-guardian"><option value="">-- select guardian --</option></select>
          <label for="child-name" style="margin-top:8px">Child name</label>
          <input id="child-name" placeholder="Child name" />
          <label for="child-dob" style="margin-top:8px">Date of birth</label>
          <input id="child-dob" type="date" />
          <div style="margin-top:12px"><button id="btn-create-child">Create Child & Generate Schedule</button></div>
          <div id="create-child-result" class="help" style="margin-top:8px"></div>
        </div>

        <hr style="margin:16px 0" />

        <h3 style="margin-top:0">Quick actions</h3>
        <div style="display:flex;gap:8px;flex-wrap:wrap">
          <button id="btn-refresh">Refresh children</button>
          <button id="btn-fetch-templates">Load vaccine templates</button>
        </div>
        <div id="templates-area" style="margin-top:12px"></div>

      </section>

      <aside class="card">
        <h3 style="margin-top:0">Children & Schedules</h3>
        <div class="muted small">Click a child to view schedules. Use the "Mark given" button on a dose to send a record to the backend.</div>
        <div style="margin-top:12px" id="children-list">Loading...</div>
      </aside>
    </div>

    <footer>
      Note: This is a frontend prototype. Adjust the API base (API_BASE) inside the script to match your backend. The prototype backend expects routes like <code>POST /api/users</code>, <code>POST /api/children</code>, and <code>GET /api/children/:id/schedule</code>. See backend canvas for full prototype details.
    </footer>
  </div>

  <script>
    // Configuration: point to your backend API base URL
    const API_BASE = window.location.origin; // change to e.g. 'http://localhost:3000' if needed

    // Utility
    function el(q){return document.querySelector(q)}
    function notify(msg, ok=true){ const n = document.createElement('div'); n.textContent = msg; n.className = ok? 'success' : 'danger'; n.style.marginTop='8px'; return n }

    // Create guardian
    el('#btn-create-guardian').addEventListener('click', async ()=>{
      const name = el('#g-name').value.trim();
      const phone = el('#g-phone').value.trim();
      const role = el('#g-role').value;
      const out = el('#create-guardian-result'); out.textContent='';
      if(!name||!phone){ out.textContent='Please provide name and phone'; return }
      try{
        const res = await fetch(API_BASE + '/api/users', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({ name, phone, role }) });
        if(!res.ok) throw new Error(await res.text());
        const data = await res.json();
        out.innerHTML='Created guardian: <strong>'+ (data.name||data.id || '') +'</strong>';
        loadGuardians();
      }catch(err){ out.textContent='Failed: '+ err.message }
    });

    // Create child
    el('#btn-create-child').addEventListener('click', async ()=>{
      const guardianId = el('#child-guardian').value;
      const name = el('#child-name').value.trim();
      const dob = el('#child-dob').value;
      const out = el('#create-child-result'); out.textContent='';
      if(!guardianId){ out.textContent='Select guardian'; return }
      if(!name||!dob){ out.textContent='Provide name and DOB'; return }
      try{
        const res = await fetch(API_BASE + '/api/children', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({ guardianId, name, dob }) });
        if(!res.ok) throw new Error(await res.text());
        const data = await res.json();
        out.innerHTML = 'Created child: <strong>' + data.child.name + '</strong> — schedules: ' + data.schedules.length;
        loadChildren();
      }catch(err){ out.textContent='Failed: '+ err.message }
    });

    // Load vaccine templates (for display)
    el('#btn-fetch-templates').addEventListener('click', async ()=>{
      const area = el('#templates-area'); area.textContent='Loading...';
      try{
        const res = await fetch(API_BASE + '/api/vaccine-templates');
        if(!res.ok) throw new Error(await res.text());
        const list = await res.json();
        area.innerHTML = '<div class="muted small">Templates loaded:</div>' + list.map(t=><div style="margin-top:8px"><strong>${t.name}</strong><div class="small">doses: ${JSON.stringify(t.doses)}</div></div>).join('');
      }catch(err){ area.textContent = 'Failed to load templates — ' + err.message }
    });

    // Load guardians into select
    async function loadGuardians(){
      try{
        const res = await fetch(API_BASE + '/api/users');
        if(!res.ok) throw new Error(await res.text());
        const data = await res.json();
        const sel = el('#child-guardian'); sel.innerHTML = '<option value="">-- select guardian --</option>';
        data.forEach(u => { const o = document.createElement('option'); o.value=u.id; o.textContent = u.name + ' ('+u.phone+')'; sel.appendChild(o) });
      }catch(err){ console.warn('Could not load guardians', err.message) }
    }

    // Load children
    async function loadChildren(){
      const list = el('#children-list'); list.innerHTML = 'Loading...';
      try{
        const res = await fetch(API_BASE + '/api/children');
        if(!res.ok) throw new Error(await res.text());
        const data = await res.json();
        if(!Array.isArray(data) || data.length===0){ list.innerHTML = '<div class="muted">No children found.</div>'; return }
        list.innerHTML = '';
        data.forEach(child => {
          const row = document.createElement('div'); row.className='child-row';
          const left = document.createElement('div'); left.innerHTML = <div><strong>${child.name}</strong> <span class="small muted">DOB: ${child.dob}</span></div><div class="small muted">ID: ${child.id}</div>;
          const right = document.createElement('div');
          const btn = document.createElement('button'); btn.textContent='View schedule'; btn.style.width='140px'; btn.addEventListener('click', ()=>{ showSchedule(child.id) });
          right.appendChild(btn);
          row.appendChild(left); row.appendChild(right);
          list.appendChild(row);
        });
      }catch(err){ list.innerHTML = '<div class="danger">Failed to load children: '+err.message+'</div>' }
    }

    // Show schedule modal (simple inline)
    async function showSchedule(childId){
      const res = await fetch(API_BASE + '/api/children/' + childId + '/schedule');
      if(!res.ok) { alert('Failed to fetch schedule: '+ await res.text()); return }
      const schedule = await res.json();
      // create a simple overlay
      const overlay = document.createElement('div'); overlay.style.position='fixed'; overlay.style.left=0; overlay.style.top=0; overlay.style.right=0; overlay.style.bottom=0; overlay.style.background='rgba(0,0,0,0.4)'; overlay.style.display='flex'; overlay.style.alignItems='center'; overlay.style.justifyContent='center';
      const box = document.createElement('div'); box.className='card'; box.style.width='720px'; box.style.maxHeight='80vh'; box.style.overflow='auto';
      box.innerHTML = <h3>Schedule for child</h3><div class='muted small'>Total doses: ${schedule.length}</div><div id='dose-list' style='margin-top:12px'></div><div style='margin-top:12px'><button id='close-schedule'>Close</button></div>;
      overlay.appendChild(box);
      document.body.appendChild(overlay);
      el('#close-schedule').addEventListener('click', ()=> overlay.remove());

      const doseList = box.querySelector('#dose-list');
      if(schedule.length===0) doseList.innerHTML = '<div class="muted">No scheduled doses.</div>';
      for(const dose of schedule){
        const d = document.createElement('div'); d.className='dose'; d.style.marginTop='8px';
        const left = document.createElement('div'); left.innerHTML = <div><strong>${dose.vaccineName}</strong> <span class='small muted'>dose ${dose.doseNumber}</span></div><div class='small'>Due: ${dose.dueDate} — status: ${dose.status}</div>;
        const right = document.createElement('div');
        const markBtn = document.createElement('button'); markBtn.textContent='Mark given'; markBtn.style.minWidth='110px'; markBtn.addEventListener('click', async ()=>{
          try{
            // Attempt to POST a vaccination record. Backend prototype route may differ — adapt as needed.
            const r = await fetch(API_BASE + '/api/children/' + childId + '/records', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({ doseScheduleId: dose.id, administeredAt: new Date().toISOString() }) });
            if(!r.ok) throw new Error(await r.text());
            alert('Marked as given');
            overlay.remove(); loadChildren();
          }catch(err){ alert('Failed to mark given: '+err.message) }
        });
        right.appendChild(markBtn);
        d.appendChild(left); d.appendChild(right); doseList.appendChild(d);
      }
    }

    // Initial load
    (async ()=>{ await loadGuardians(); await loadChildren(); })();

    // Refresh button
    el('#btn-refresh').addEventListener('click', ()=>{ loadGuardians(); loadChildren(); });
  </script>
</body>
</html>
