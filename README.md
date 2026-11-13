# Anleitung: OKX-Kreditzinsen automatisch erfassen und anzeigen

Stand: 13.11.2026

## 1. Überblick

Dieses System erfasst automatisch die Zinsen, die OKX Europe für deinen USDC-Kredit berechnet, und zeigt sie in einem Dashboard an.

**Bestandteile:**

| Bestandteil | Wo | Aufgabe |
|---|---|---|
| API-Schlüssel | OKX | Erlaubt das Auslesen deiner Zinsdaten (nur Leserechte) |
| Repository „Crypto“ (privat) | GitHub | Enthält das Skript, den Workflow und die gesammelten Daten |
| Skript `okx_zinsen.py` | Repository „Crypto“ | Holt die Zinsdaten von OKX und speichert sie als CSV |
| Workflow `zinsen.yml` | Repository „Crypto“ | Startet das Skript automatisch einmal täglich |
| Token „Zins-Dashboard“ | GitHub | Erlaubt dem Dashboard, die Daten aus dem privaten Repository zu lesen |
| Repository „zins-dashboard“ (öffentlich) | GitHub | Enthält nur das Dashboard als Webseite |

**Ablauf:** Jeden Morgen startet GitHub das Skript. Es holt alle neuen Stundenwerte von OKX und speichert sie in der Datei `zinsen_stuendlich.csv`. Wenn du das Dashboard öffnest, lädt es diese Datei und zeigt Zinsen, Kosten und Verlauf an.

**Voraussetzungen:** ein OKX-Europe-Konto mit laufendem USDC-Kredit und ein kostenloses GitHub-Konto.

---

## 2. Schritt 1: API-Schlüssel bei OKX erstellen

1. Bei OKX einloggen und im Profilmenü den Bereich **API** öffnen.
2. Einen neuen API-Schlüssel erstellen.
3. Einen Namen vergeben, z. B. „Zinsabruf“.
4. Eine **Passphrase** festlegen. Das ist ein eigenes Passwort nur für diesen Schlüssel, nicht dein Login-Passwort. Notiere sie dir.
5. Bei den Berechtigungen **nur „Lesen“** aktivieren. Kein Handel, keine Abhebung.
6. Das Feld für die IP-Beschränkung **leer lassen**, weil GitHub wechselnde IP-Adressen nutzt.
7. Erstellen und die angezeigten Werte sicher notieren:
   - **API Key**
   - **Secret Key** (wird meist nur einmal angezeigt)
   - **Passphrase** (selbst festgelegt)

Wenn der Secret Key verloren geht: Schlüssel löschen und einen neuen erstellen.

---

## 3. Schritt 2: Privates Repository anlegen und Skript hochladen

### 3.1 Repository anlegen

1. Auf github.com oben rechts auf **+** → **New repository**.
2. Name: **Crypto**
3. Sichtbarkeit: **Private** (wichtig, sonst wären deine Daten öffentlich).
4. README: optional. .gitignore: keine oder „Python“. Lizenz: **None**.
5. **Create repository**.

### 3.2 Skript erstellen

1. Im Repository auf **Add file** → **Create new file**.
2. Dateiname: **`okx_zinsen.py`** (direkt im Hauptordner, nicht in einem Unterordner).
3. Folgenden Code einfügen:

```python
import base64, csv, hmac, json, os, time
from datetime import datetime, timezone
from hashlib import sha256
from collections import defaultdict
import requests

# --- Einstellungen ---
API_KEY = os.environ.get("OKX_KEY", "")
API_SECRET = os.environ.get("OKX_SECRET", "")
PASSPHRASE = os.environ.get("OKX_PASS", "")
BASE_URL = "https://eea.okx.com"   # falls Fehler: "https://www.okx.com" probieren
CCY = "USDC"
ORDNER = os.path.dirname(os.path.abspath(__file__))
STUNDEN_CSV = os.path.join(ORDNER, "zinsen_stuendlich.csv")
DASHBOARD_TXT = os.path.join(ORDNER, "dashboard_import.txt")

def signiert_get(pfad):
    ts = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%f")[:-3] + "Z"
    sig = base64.b64encode(hmac.new(API_SECRET.encode(), (ts + "GET" + pfad).encode(), sha256).digest()).decode()
    headers = {"OK-ACCESS-KEY": API_KEY, "OK-ACCESS-SIGN": sig,
               "OK-ACCESS-TIMESTAMP": ts, "OK-ACCESS-PASSPHRASE": PASSPHRASE}
    r = requests.get(BASE_URL + pfad, headers=headers, timeout=20)
    r.raise_for_status()
    daten = r.json()
    if daten.get("code") != "0":
        raise RuntimeError(f"OKX-Fehler: {daten}")
    return daten["data"]

def vorhandene_zeilen():
    if not os.path.exists(STUNDEN_CSV):
        return {}
    with open(STUNDEN_CSV, newline="", encoding="utf-8") as f:
        return {z["ts"]: z for z in csv.DictReader(f)}

def neue_daten_holen(bekannt):
    neu, after = [], ""
    while True:
        pfad = f"/api/v5/account/interest-accrued?ccy={CCY}&limit=100" + (f"&after={after}" if after else "")
        seite = signiert_get(pfad)
        if not seite:
            break
        stop = False
        for e in seite:
            if e["ts"] in bekannt:
                stop = True
                break
            neu.append(e)
        if stop or len(seite) < 100:
            break
        after = seite[-1]["ts"]
        time.sleep(0.3)
    return neu

def main():
    bekannt = vorhandene_zeilen()
    for e in neue_daten_holen(bekannt):
        stunde_rate = float(e["interestRate"])
        bekannt[e["ts"]] = {
            "ts": e["ts"],
            "zeit": datetime.fromtimestamp(int(e["ts"]) / 1000).strftime("%Y-%m-%d %H:%M"),
            "kredit_usdc": e.get("liab", ""),
            "zinsen_usdc": e.get("interest", ""),
            "stundenzins_dezimal": e["interestRate"],
            "jahreszins_prozent": f"{stunde_rate * 8760 * 100:.4f}",
        }
    zeilen = sorted(bekannt.values(), key=lambda z: int(z["ts"]))
    if not zeilen:
        print("Keine Zinsdaten gefunden.")
        return
    with open(STUNDEN_CSV, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=list(zeilen[0].keys()))
        w.writeheader()
        w.writerows(zeilen)
    tage = defaultdict(list)
    for z in zeilen:
        tage[z["zeit"][:10]].append(float(z["jahreszins_prozent"]))
    with open(DASHBOARD_TXT, "w", encoding="utf-8") as f:
        for tag in sorted(tage):
            f.write(f"{tag};{sum(tage[tag]) / len(tage[tag]):.4f}\n")
    letzte = zeilen[-1]
    print(f"{len(zeilen)} Stunden gespeichert. Letzter Wert: {letzte['zeit']} -> {float(letzte['jahreszins_prozent']):.2f} % p. a.")

if __name__ == "__main__":
    main()
```

4. **Commit changes** klicken.

---

## 4. Schritt 3: Secrets und Berechtigungen einrichten

### 4.1 Secrets anlegen

Im Repository „Crypto“: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**. Drei Secrets anlegen:

| Name (exakt so schreiben) | Wert |
|---|---|
| `OKX_KEY` | API Key von OKX |
| `OKX_SECRET` | Secret Key von OKX |
| `OKX_PASS` | Passphrase, die du bei OKX festgelegt hast |

Werte ohne Leerzeichen oder Anführungszeichen einfügen.

### 4.2 Schreibrechte für den Workflow

**Settings** → **Actions** → **General** → Abschnitt **Workflow permissions** → **Read and write permissions** auswählen → **Save**.

Ohne diese Einstellung kann der Workflow die gesammelten Daten nicht im Repository speichern.

---

## 5. Schritt 4: Workflow anlegen und testen

### 5.1 Workflow-Datei erstellen

1. Im Repository auf den Reiter **Actions**.
2. Auf **set up a workflow yourself →** klicken.
3. Den Dateinamen oben zu **`zinsen.yml`** ändern. Der Pfad muss **`.github/workflows/zinsen.yml`** lauten.
4. Den Beispielinhalt löschen und folgenden Code einfügen:

```yaml
name: OKX Zinsen abrufen

on:
  schedule:
    - cron: "15 5 * * *"   # täglich um 05:15 UTC
  workflow_dispatch:         # zusätzlich manuell startbar

jobs:
  abrufen:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install requests
      - run: python okx_zinsen.py
        env:
          OKX_KEY: ${{ secrets.OKX_KEY }}
          OKX_SECRET: ${{ secrets.OKX_SECRET }}
          OKX_PASS: ${{ secrets.OKX_PASS }}
      - name: Daten speichern
        run: |
          git config user.name "zins-bot"
          git config user.email "zins-bot@users.noreply.github.com"
          git add zinsen_stuendlich.csv dashboard_import.txt
          git commit -m "Zinsdaten aktualisiert" || echo "Keine Änderungen"
          git push
```

5. **Commit changes** → nochmal **Commit changes**.

**Häufiger Fehler:** Liegt der Ordner `workflows` direkt im Hauptordner statt in `.github`, erkennt GitHub den Workflow nicht. Korrektur: Datei bearbeiten, im Dateinamenfeld mit der Rücktaste den Ordnerpfad bearbeitbar machen und `.github/` davor setzen.

### 5.2 Zeitplan

Die Zeile `cron: "15 5 * * *"` bedeutet: täglich um 05:15 UTC, also 6:15 Uhr deutscher Winterzeit bzw. 7:15 Uhr Sommerzeit. Die erste Zahl ist die Minute, die zweite die Stunde in UTC. Beispiel für 20:00 Uhr Winterzeit: `"0 19 * * *"`.

Einmal täglich reicht, weil das Skript alle fehlenden Stunden rückwirkend nachholt.

### 5.3 Manuell testen

1. Reiter **Actions** → links **OKX Zinsen abrufen** → rechts **Run workflow** → **Run workflow**.
2. Nach etwa 20 Sekunden erscheint ein Lauf. Grüner Haken = erfolgreich.
3. Auf den Lauf → **abrufen** → Schritt **Run python okx_zinsen.py** aufklappen.
4. Dort sollte stehen: **„… Stunden gespeichert. Letzter Wert: … → … % p. a.“**
5. Den Jahreszins mit der OKX-Seite „Margin-Zinssatz“ vergleichen. Er sollte ungefähr übereinstimmen.
6. Unter **Code** müssen jetzt die Dateien `zinsen_stuendlich.csv` und `dashboard_import.txt` vorhanden sein.

Die Hinweise zu Node.js und Ubuntu unter „Annotations“ sind harmlos. Wer sie loswerden will, ersetzt `actions/checkout@v4` durch `@v5` und `actions/setup-python@v5` durch `@v6`.

---

## 6. Schritt 5: GitHub-Token für das Dashboard erstellen

1. Oben rechts auf das Profilbild → **Settings** → ganz unten links **Developer settings**.
2. **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
3. Name: „Zins-Dashboard“, Ablauf: z. B. 1 Jahr.
4. **Repository access:** „Only select repositories“ → **Crypto**.
5. **Permissions** → Repository permissions → **Contents: Read-only**. Sonst nichts.
6. **Generate token** und den Token kopieren. Er wird nur einmal angezeigt.

Wenn der Token abläuft, zeigt das Dashboard „Token ungültig oder abgelaufen“. Dann einfach einen neuen erstellen und eintragen.

---

## 7. Schritt 6: Dashboard veröffentlichen

### 7.1 Öffentliches Repository für das Dashboard

1. Neues Repository: Name **`zins-dashboard`**, Sichtbarkeit **Public**.
2. **Add file** → **Create new file**, Dateiname **`index.html`**.
3. Folgenden Code einfügen:

```html
<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>OKX USDC-Zins-Dashboard</title>
<style>
:root{--bg:#fff;--card:#f5f4f0;--text:#1f1e1d;--muted:#6b6a66;--border:#e3e1da;--danger:#c0392b;--ok:#3b6d11}
@media(prefers-color-scheme:dark){:root{--bg:#1f1e1d;--card:#2a2928;--text:#f0efe9;--muted:#a3a29c;--border:#3a3937;--danger:#f08080;--ok:#97c459}}
body{font-family:system-ui,sans-serif;background:var(--bg);color:var(--text);max-width:820px;margin:2rem auto;padding:0 1rem}
h1{font-size:22px;font-weight:500;margin:0}
.top{display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:.5rem}
.g{display:grid;grid-template-columns:repeat(auto-fit,minmax(170px,1fr));gap:12px}
.m{background:var(--card);border-radius:8px;padding:1rem}
.ml{font-size:13px;color:var(--muted);margin:0}.mv{font-size:22px;font-weight:500;margin:4px 0 0}
label{font-size:13px;color:var(--muted);display:block;margin-bottom:4px}
input,button{font:inherit;color:var(--text);background:var(--bg);border:1px solid var(--border);border-radius:8px;padding:6px 10px;box-sizing:border-box}
input{width:100%;height:36px}button{cursor:pointer}button:hover{background:var(--card)}
table{width:100%;border-collapse:collapse;font-size:14px}
th,td{padding:8px 6px;border-bottom:1px solid var(--border);text-align:right}
th{font-weight:500;color:var(--muted);font-size:13px}th:first-child,td:first-child{text-align:left}
.status{font-size:13px;color:var(--muted);margin:0 0 1.5rem}.err{color:var(--danger)}
details{margin-top:2rem;border-top:1px solid var(--border);padding-top:1rem}
summary{cursor:pointer;color:var(--muted);font-size:14px}
</style>
</head>
<body>
<div class="top"><h1>OKX USDC-Zins-Dashboard</h1><button id="reload">Neu laden</button></div>
<p class="status" id="status">Lade Daten…</p>

<div class="g" style="margin-bottom:1.5rem">
<div class="m"><p class="ml">Aktueller Zins p. a.</p><p class="mv" id="s1">–</p></div>
<div class="m"><p class="ml">Ø letzte 7 Tage p. a.</p><p class="mv" id="s2">–</p></div>
<div class="m"><p class="ml">Kosten/Jahr (Ø 7 Tage)</p><p class="mv" id="s3">–</p></div>
<div class="m"><p class="ml">Gezahlte Zinsen gesamt</p><p class="mv" id="s4">–</p></div>
</div>

<div style="position:relative;height:240px;margin-bottom:1.5rem"><canvas id="c"></canvas></div>

<table>
<thead><tr><th>Datum</th><th>Ø Zins p. a.</th><th>Min–Max</th><th>Gezahlt</th><th>Kredit</th></tr></thead>
<tbody id="tb"></tbody>
</table>

<details id="cfg">
<summary>Einstellungen</summary>
<div class="g" style="margin-top:1rem">
<div><label for="owner">GitHub-Benutzername</label><input id="owner" placeholder="Kirsten281178"></div>
<div><label for="repo">Repository</label><input id="repo" placeholder="Crypto"></div>
<div style="grid-column:1/-1"><label for="token">Token (nur Lesezugriff)</label><input id="token" type="password" placeholder="github_pat_…"></div>
</div>
<div style="margin-top:12px"><button id="saveCfg">Speichern und laden</button> <button id="delCfg">Token löschen</button></div>
</details>

<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<script>
const KEY='okxZinsDashboardCfg';
const $=i=>document.getElementById(i);
const f=(n,d=2)=>n.toLocaleString('de-DE',{minimumFractionDigits:d,maximumFractionDigits:d});
let cfg={owner:'Kirsten281178',repo:'Crypto',token:''};
try{const s=JSON.parse(localStorage.getItem(KEY));if(s)cfg={...cfg,...s};}catch(e){}
$('owner').value=cfg.owner;$('repo').value=cfg.repo;$('token').value=cfg.token;
let ch=null;

function status(t,err){$('status').textContent=t;$('status').className='status'+(err?' err':'');}

function parseCsv(text){
  const lines=text.trim().split(/\r?\n/);
  const head=lines.shift().split(',');
  const ix=n=>head.indexOf(n);
  return lines.map(l=>{const c=l.split(',');return{
    ts:+c[ix('ts')],
    liab:parseFloat(c[ix('kredit_usdc')])||0,
    int:parseFloat(c[ix('zinsen_usdc')])||0,
    pa:parseFloat(c[ix('jahreszins_prozent')])
  };}).filter(r=>r.ts&&!isNaN(r.pa)).sort((a,b)=>a.ts-b.ts);
}

function render(rows){
  if(!rows.length){status('Die Datei enthält noch keine Daten.',true);return;}
  const last=rows[rows.length-1];
  const w=rows.slice(-168);
  const avg7=w.reduce((s,r)=>s+r.pa,0)/w.length;
  const total=rows.reduce((s,r)=>s+r.int,0);
  $('s1').textContent=f(last.pa)+' %';
  $('s2').textContent=f(avg7)+' %';
  $('s3').textContent=f(last.liab*avg7/100,0)+' USDC';
  $('s4').textContent=f(total)+' USDC';

  const tage={};
  rows.forEach(r=>{
    const d=new Date(r.ts).toLocaleDateString('sv-SE');
    (tage[d]=tage[d]||[]).push(r);
  });
  const liste=Object.keys(tage).sort().map(d=>{
    const t=tage[d],p=t.map(r=>r.pa);
    return{d,avg:p.reduce((a,b)=>a+b,0)/p.length,min:Math.min(...p),max:Math.max(...p),
      int:t.reduce((s,r)=>s+r.int,0),liab:t[t.length-1].liab,n:t.length};
  });

  $('tb').innerHTML=liste.slice().reverse().map(t=>
    `<tr><td>${t.d.split('-').reverse().join('.')}${t.n<24?' <span style="color:var(--muted);font-size:12px">('+t.n+' h)</span>':''}</td>
    <td>${f(t.avg)} %</td><td>${f(t.min,1)}–${f(t.max,1)} %</td>
    <td>${f(t.int)} USDC</td><td>${f(t.liab,0)}</td></tr>`).join('');

  const zeit=new Date(last.ts).toLocaleString('de-DE',{dateStyle:'short',timeStyle:'short'});
  status(`Stand: ${zeit} Uhr · ${rows.length} Stunden · ${liste.length} Tage`);

  if(typeof Chart==='undefined')return;
  const data={labels:liste.map(t=>t.d.slice(8,10)+'.'+t.d.slice(5,7)),
    datasets:[{data:liste.map(t=>+t.avg.toFixed(2)),borderColor:'#378ADD',backgroundColor:'#378ADD',tension:0.2,pointRadius:3}]};
  if(ch){ch.data=data;ch.update();return;}
  ch=new Chart($('c'),{type:'line',data,options:{responsive:true,maintainAspectRatio:false,
    plugins:{legend:{display:false},tooltip:{callbacks:{label:c=>f(c.parsed.y)+' % p. a.'}}},
    scales:{y:{ticks:{color:'#888',callback:v=>v.toFixed(1)+' %'},grid:{color:'rgba(128,128,128,0.15)'}},
            x:{ticks:{color:'#888'},grid:{display:false}}}}});
}

async function laden(){
  if(!cfg.token||!cfg.owner||!cfg.repo){
    $('cfg').open=true;status('Trag unten in den Einstellungen deinen GitHub-Token ein.',true);return;}
  status('Lade Daten…');
  try{
    const url=`https://api.github.com/repos/${encodeURIComponent(cfg.owner)}/${encodeURIComponent(cfg.repo)}/contents/zinsen_stuendlich.csv`;
    const r=await fetch(url,{cache:'no-store',headers:{
      'Authorization':'Bearer '+cfg.token,
      'Accept':'application/vnd.github.raw+json',
      'X-GitHub-Api-Version':'2022-11-28'}});
    if(r.status===401){status('Token ungültig oder abgelaufen. Bitte neuen Token eintragen.',true);$('cfg').open=true;return;}
    if(r.status===404){status('Datei oder Repository nicht gefunden. Prüf Benutzername, Repository und Token-Rechte.',true);$('cfg').open=true;return;}
    if(!r.ok){status('Fehler beim Laden (Status '+r.status+').',true);return;}
    render(parseCsv(await r.text()));
  }catch(e){status('Keine Verbindung zu GitHub. Bist du online?',true);}
}

$('saveCfg').onclick=()=>{
  cfg={owner:$('owner').value.trim(),repo:$('repo').value.trim(),token:$('token').value.trim()};
  try{localStorage.setItem(KEY,JSON.stringify(cfg));}catch(e){}
  $('cfg').open=false;laden();
};
$('delCfg').onclick=()=>{
  cfg.token='';$('token').value='';
  try{localStorage.setItem(KEY,JSON.stringify(cfg));}catch(e){}
  status('Token gelöscht.',true);
};
$('reload').onclick=laden;
laden();
</script>
</body>
</html>
```

4. **Commit changes**.

### 7.2 GitHub Pages aktivieren

1. Im Repository „zins-dashboard“: **Settings** → **Pages**.
2. Source: **Deploy from a branch**, Branch: **main**, Ordner: **/ (root)** → **Save**.
3. Nach ein bis zwei Minuten ist das Dashboard erreichbar unter:
   **https://kirsten281178.github.io/zins-dashboard/**

### 7.3 Dashboard einrichten

1. Die Adresse im Browser öffnen.
2. Unten öffnen sich automatisch die **Einstellungen**. Benutzername (Kirsten281178) und Repository (Crypto) sind vorausgefüllt.
3. Den Token aus Schritt 5 einfügen → **Speichern und laden**.
4. Auf jedem weiteren Gerät (z. B. Handy) einmalig den Token eintragen.

---

## 8. Nutzung, Wartung und Fehlerbehebung

### 8.1 Was das Dashboard zeigt

| Anzeige | Bedeutung |
|---|---|
| Aktueller Zins p. a. | Jahreszins der zuletzt erfassten Stunde |
| Ø letzte 7 Tage p. a. | Durchschnitt der letzten 168 Stunden |
| Kosten/Jahr (Ø 7 Tage) | Hochrechnung mit aktuellem Kreditbetrag und 7-Tage-Durchschnitt |
| Gezahlte Zinsen gesamt | Summe der tatsächlich berechneten Zinsen im erfassten Zeitraum |
| Tabelle | Pro Tag: Durchschnittszins, Spanne, gezahlte Zinsen, Kreditbetrag |

Tage mit weniger als 24 Stunden Daten sind in der Tabelle gekennzeichnet. Zeiten werden automatisch in deutsche Zeit umgerechnet. In den CSV-Dateien selbst stehen die Zeiten in UTC.

### 8.2 Umrechnung Stundenzins → Jahreszins

Jahreszins = Stundenzins × 8.760
Beispiel: 0,001085 % pro Stunde × 8.760 ≈ 9,5 % pro Jahr

### 8.3 Regelmäßige Kontrolle

- Unter **Actions** im Repository „Crypto“ sollte jeden Morgen ein Lauf mit „scheduled“ erscheinen.
- Unter **Code** sollte täglich ein Commit „Zinsdaten aktualisiert“ auftauchen.
- GitHub startet geplante Läufe manchmal einige Minuten verspätet. Das ist normal.

### 8.4 Fehlerbehebung

| Problem | Ursache und Lösung |
|---|---|
| Kein „Run workflow“-Knopf | `zinsen.yml` liegt nicht in `.github/workflows/`. Pfad korrigieren. |
| Workflow rot, Fehler „OKX-Fehler“ | Secrets falsch geschrieben oder Werte falsch. Namen und Werte prüfen. |
| Fehler bei der Verbindung zu OKX | In `okx_zinsen.py` `BASE_URL` auf `https://www.okx.com` ändern. |
| „Keine Zinsdaten gefunden“ | Aktuell kein Kredit offen oder API liefert nichts. Bei Bedarf OKX-Support fragen. |
| Fehler beim Schritt „Daten speichern“ | Workflow-Schreibrechte fehlen (Schritt 3, Abschnitt 4.2). |
| Dashboard: „Token ungültig oder abgelaufen“ | Neuen Token erstellen (Schritt 5) und eintragen. |
| Dashboard: „Datei oder Repository nicht gefunden“ | Benutzername, Repository-Name und Token-Berechtigung (Contents: Read-only auf „Crypto“) prüfen. |
| Grafik fehlt, Tabelle da | Keine Internetverbindung zum Grafik-Dienst. Seite neu laden. |

### 8.5 Sicherheitshinweise

- Der OKX-API-Schlüssel darf **nur Leserechte** haben, niemals Handel oder Abhebung.
- Das Repository „Crypto“ muss **privat** bleiben.
- Den GitHub-Token **nie** in die Datei `index.html` schreiben. Er wird nur im Browser gespeichert.
- Auf fremden oder geteilten Geräten den Token nicht eintragen oder danach mit **Token löschen** entfernen.
- Bei Verdacht auf Missbrauch: API-Schlüssel bei OKX löschen und Token bei GitHub widerrufen (Settings → Developer settings → Personal access tokens).
