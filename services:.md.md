import datetime
import csv
import os
import urllib.request
import xml.etree.ElementTree as ET
from fastapi import FastAPI, Depends, Form, UploadFile, File, HTTPException, status
from fastapi.responses import HTMLResponse, RedirectResponse, StreamingResponse
from fastapi.security import HTTPBasic, HTTPBasicCredentials
from sqlalchemy import create_engine, Column, Integer, String, Date, ForeignKey, extract, text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
import io
import secrets

# =====================================================================
# MOTORE DATABASE SOSTY (VERSIONE ORIGINALE E LEGGERA)
# =====================================================================
DATABASE_URL = "sqlite:///./database.db"
engine_db = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine_db)
Base = declarative_base()

class Docente(Base):
    __tablename__ = "docenti"
    id = Column(Integer, primary_key=True, autoincrement=True)
    nominativo = Column(String, unique=True, index=True)
    tipo = Column(String) 

class OrarioSettimanale(Base):
    __tablename__ = "orari_completi"
    id = Column(Integer, primary_key=True, autoincrement=True)
    docente_id = Column(Integer, ForeignKey("docenti.id"))
    giorno_settimana = Column(String)  
    ora_lezione = Column(Integer)      
    classe_assegnata = Column(String)  

class Sostituzione(Base):
    __tablename__ = "sostituzioni_giornaliere"
    id = Column(Integer, primary_key=True, autoincrement=True)
    data = Column(Date, index=True) 
    ora = Column(Integer)
    classe = Column(String)
    assente_id = Column(Integer, ForeignKey("docenti.id"))   
    sostituto_id = Column(Integer, ForeignKey("docenti.id"), nullable=True) 
    timestamp_aggiornamento = Column(String, default=lambda: datetime.datetime.now().strftime("%H:%M:%S"))

class ImpostazioniScuola(Base):
    __tablename__ = "impostazioni"
    id = Column(Integer, primary_key=True)
    chiave = Column(String, unique=True)
    valore = Column(String)

Base.metadata.create_all(bind=engine_db)

def get_db():
    db = SessionLocal()
    try: yield db
    finally: db.close()

def recupera_news_lato_server():
    try:
        url = "https://www.adnkronos.com/rss/all"
        req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req, timeout=3) as response:
            xml_data = response.read()
        root = ET.fromstring(xml_data)
        titoli = []
        for item in root.findall('.//item')[:8]: 
            titolo = item.find('title')
            if titolo is not None and titolo.text:
                titoli.append(titolo.text.strip())
        if titoli:
            return " | NOTIZIE NAZIONALI: " + " ••• ".join(titoli)
    except Exception:
        pass
    return ""

app = FastAPI()

# =====================================================================
# SISTEMA DI PROTEZIONE PASSWORD (HTTP BASIC AUTH)
# =====================================================================
security = HTTPBasic()

ADMIN_USER = os.getenv("SOSTY_USER", "admin")
ADMIN_PASSWORD = os.getenv("SOSTY_PASSWORD", "sosty2026")

def verifica_credenziali(credentials: HTTPBasicCredentials = Depends(security)):
    corretto_user = secrets.compare_digest(credentials.username, ADMIN_USER)
    corretto_pass = secrets.compare_digest(credentials.password, ADMIN_PASSWORD)
    if not (corretto_user and corretto_pass):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenziali Sosty non valide",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username

# =====================================================================
# STILI CSS GENERALI (SOSTY STYLE ORIGINAL)
# =====================================================================
STILE_ITALIA = """
<link href="https://fonts.googleapis.com/css2?family=Titillium+Web:wght@300;400;600;700&display=swap" rel="stylesheet">
<style>
    :root { 
        --blue-italia: #0066cc; 
        --blue-dark: #00264d; 
        --blue-light: #f0f6fc;
        --blue-hover: #0052a3;
        --gray-bg: #f4f5f7; 
        --text-dark: #1c2024;
        --success: #008758; 
        --error: #c52d3a; 
        --border: #d2d6da;
        --warning: #ffab00;
        --assistente-giallo: #ffeb3b;
        --assistente-testo: #333300;
    }
    body { font-family: 'Titillium Web', sans-serif; background-color: var(--gray-bg); margin: 0; padding: 0; color: var(--text-dark); }
    .navbar { background: var(--blue-italia); color: white; padding: 15px 30px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
    .navbar h1 { margin: 0; font-size: 22px; text-transform: uppercase; font-weight: 600; letter-spacing: 0.5px; }
    .container { max-width: 1250px; margin: 20px auto; padding: 0 15px; display: grid; grid-template-columns: 1fr 1.2fr; gap: 20px; }
    .full-width { grid-column: span 2; }
    .card { background: white; border-radius: 4px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); padding: 20px; border-top: 4px solid var(--blue-italia); margin-bottom: 15px; }
    .card h2 { margin-top: 0; color: var(--blue-dark); font-size: 19px; border-bottom: 1px solid #ddd; padding-bottom: 8px; text-transform: uppercase; font-weight: 600; }
    label { font-weight: 600; display: block; margin-bottom: 4px; color: var(--blue-dark); margin-top: 10px; }
    select, input, textarea { width: 100%; padding: 8px; border: 1px solid var(--border); border-radius: 4px; font-size: 15px; box-sizing: border-box; background-color: #fff; font-family: inherit; }
    .btn { background: var(--blue-italia); color: white; border: none; padding: 10px 20px; border-radius: 4px; font-weight: 600; cursor: pointer; text-transform: uppercase; font-size: 12px; transition: 0.2s; text-decoration: none; display: inline-block; }
    .btn:hover { background: var(--blue-hover); }
    .btn-danger { background: var(--error); }
    .btn-danger:hover { background: #96222b; }
    .btn-sm { padding: 6px 12px; font-size: 11px; font-weight: 600; }
    .btn-xs { padding: 4px 8px; font-size: 11px; background: var(--error); color: white; border: none; border-radius: 3px; cursor: pointer; font-weight: 600; text-transform: uppercase; display: inline-block; }
    .btn-xs:hover { background: #96222b; }
    .btn-success { background: var(--success); }
    table { width: 100%; border-collapse: collapse; margin-top: 10px; background: white; }
    th { background: var(--blue-light); color: var(--blue-dark); text-align: left; padding: 10px; border-bottom: 2px solid var(--border); font-size: 13px; text-transform: uppercase; font-weight: 600; }
    td { padding: 10px 12px; border-bottom: 1px solid #eee; font-size: 15px; vertical-align: middle; }
    
    .badge-ok { background: #d1e7dd; color: #0f5132; padding: 6px 12px; border-radius: 4px; font-weight: 600; display: inline-block; }
    .badge-ko { background: #f8d7da; color: #842029; padding: 6px 12px; border-radius: 4px; font-weight: 600; display: inline-block; border: 1px solid var(--error); }
    .badge-assistente { background: var(--assistente-giallo); color: var(--assistente-testo); padding: 6px 12px; border-radius: 4px; font-weight: 600; display: inline-block; border: 1px solid #fbc02d; }
    
    .select-tabella { width: auto; padding: 8px 12px; font-size: 15px; font-weight: 600; margin-right: 5px; display: inline-block; color: var(--blue-dark); border: 1px solid var(--blue-italia); border-radius: 4px; cursor: pointer; }
    .alert { background: #d1e7dd; color: #0f5132; padding: 12px; border-radius: 4px; border: 1px solid #badbcc; font-weight: 600; margin-bottom: 15px; }
    .date-navigator { background: var(--blue-light); padding: 12px; border-radius: 4px; border: 1px solid var(--border); display: flex; align-items: center; justify-content: space-between; margin-bottom: 15px; }
</style>
<script>
    function toggleCampiAssenza() {
        var tipo = document.getElementById("tipo_assenza").value;
        document.getElementById("box_ora_singola").style.display = (tipo === "GIORNATA") ? "none" : "block";
    }
    function cambiaDataFiltro() {
        var dataSelezionata = document.getElementById("data_visualizzata").value;
        window.location.href = "/?data=" + dataSelezionata;
    }
    function salvaSalvataggioAutomatico(formId) {
        document.getElementById(formId).submit();
    }
    function invioImmediatoAssenza() {
        var selectDocente = document.getElementById("docente_assente_select");
        if(selectDocente.value !== "") {
            document.getElementById("form-registra-assenza").submit();
        }
    }
</script>
"""

@app.get("/", response_class=HTMLResponse)
def dashboard(data: str = None, msg: str = None, db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    if not data:
        data_lavoro = datetime.date.today()
    else:
        data_lavoro = datetime.date.fromisoformat(data)
        
    str_data_lavoro = data_lavoro.isoformat()
    id_assenti_oggi = [s.assente_id for s in db.query(Sostituzione).filter(Sostituzione.data == data_lavoro).all()]
    
    docenti_disponibili_menu = db.query(Docente).filter(Docente.tipo != "assistente", Docente.id.not_in(id_assenti_oggi)).order_by(Docente.nominativo).all()
    tutti_i_docenti = db.query(Docente).order_by(Docente.nominativo).all()
    sostituzioni = db.query(Sostituzione).filter(Sostituzione.data == data_lavoro).order_by(Sostituzione.ora).all()
    
    avviso_db = db.query(ImpostazioniScuola).filter(ImpostazioniScuola.chiave == "avviso_totem").first()
    testo_avviso = avviso_db.valore if avviso_db else ""

    opzioni_docenti = "<option value='' disabled selected>-- SELEZIONA UN DOCENTE --</option>"
    opzioni_docenti += "".join([f"<option value='{d.id}'>{d.nominativo.upper()} ({d.tipo.upper()})</option>" for d in docenti_disponibili_menu])
    
    righe_tabella = ""
    for s in sostituzioni:
        assente = db.query(Docente).filter(Docente.id == s.assente_id).first()
        opzioni_sostituto = f"<option value=''>-- SELEZIONA A MANO --</option>"
        
        for d in tutti_i_docenti:
            if d.id != s.assente_id and d.id not in id_assenti_oggi:
                selezionato = "selected" if s.sostituto_id == d.id else ""
                opzioni_sostituto += f"<option value='{d.id}' {selezionato}>{d.nominativo.upper()} ({d.tipo.upper()})</option>"
        
        form_id = f"form_sost_{s.id}"
        form_assegnazione = f"""
        <form id="{form_id}" action="/assegna-manuale" method="post" style="margin:0; display:inline-block;">
            <input type="hidden" name="sostituzione_id" value="{s.id}">
            <input type="hidden" name="data_corrente" value="{str_data_lavoro}">
            <select name="sostituto_id" class="select-tabella" onchange="salvaSalvataggioAutomatico('{form_id}')">{opzioni_sostituto}</select>
        </form>
        """
        
        righe_tabella += f"""
        <tr>
            <td>{s.ora}° Ora</td>
            <td>{s.classe}</td>
            <td>{assente.nominativo.upper() if assente else "SCONOSCIUTO"}</td>
            <td>
                <div style="display:flex; align-items:center; justify-content:space-between; gap:10px;">
                    {form_assegnazione}
                    <a href="/elimina-voce?id={id}&data={str_data_lavoro}" class="btn-xs" style="text-decoration:none;">Elimina</a>
                </div>
            </td>
        </tr>
        """

    banner_notifica = f"<div class='alert'>{msg}</div>" if msg else ""

    if not tutti_i_docenti:
        pannello_orario = """
        <div class="card full-width">
            <h2>1. Carica il Modello Orario d'Istituto (.csv)</h2>
            <p>Seleziona il tuo file <strong>orario.csv</strong> per configurare l'anagrafica dei docenti.</p>
            <form action="/import-motore" method="post" enctype="multipart/form-data">
                <input type="file" name="file" accept=".csv" required>
                <br><br>
                <button type="submit" class="btn">IMPORTA ORARIO</button>
            </form>
        </div>
        """
    else:
        pannello_orario = f"""
        <div class="card full-width" style="border-top-color: var(--success); background: #fafafa;">
            <p style="margin:0; font-weight:600; color: var(--success); font-size:16px; display:flex; justify-content:space-between; align-items:center;">
                <span>✓ Sosty Pronto: Caricati {len(tutti_i_docenti)} record.</span>
                <a href="/reset-totale" class="btn btn-danger" style="font-size:11px; padding:6px 12px;">Sostituisci File Orario</a>
            </p>
        </div>
        """

    return f"""
    <!DOCTYPE html><html><head>{STILE_ITALIA}<title>Sosty - Amministrazione</title></head>
    <body>
        <div class="navbar"><h1>Sosty - Pannello Amministrazione e Logistica</h1></div>
        <div class="container">
            <div class="full-width">{banner_notifica}</div>
            {pannello_orario}
            
            <div class="full-width date-navigator">
                <span>Pianifica data specifica:</span>
                <input type="date" id="data_visualizzata" value="{str_data_lavoro}" onchange="cambiaDataFiltro()" style="width: 200px;">
            </div>
            
            <div class="card">
                <h2>2. Registra Assenza Personale</h2>
                <form id="form-registra-assenza" action="/elabora-assenza" method="post">
                    <input type="hidden" name="data_assenza" value="{str_data_lavoro}">
                    <label>Estensione Assenza:</label>
                    <select name="tipo_assenza" id="tipo_assenza" onchange="toggleCampiAssenza()">
                        <option value="GIORNATA">Intera Giornata</option>
                        <option value="ORA_SINGOLA">Ora Singola / Permesso Breve</option>
                    </select>
                    <div id="box_ora_singola" style="display:none; margin-top:10px;">
                        <label>Inserisci le ore del permesso (separate da virgola, es: 1,2 o 3,4,5):</label>
                        <input type="text" name="ore_inserite" placeholder="es: 1,2,3" style="text-transform: uppercase;">
                    </div>
                    <label>Seleziona chi manca (Registra all'istante):</label>
                    <select name="docente_id" id="docente_assente_select" onchange="invioImmediatoAssenza()" required>{opzioni_docenti}</select>
                </form>
                
                <hr style="margin-top:20px; border:0; border-top:1px solid #eee;">
                
                <h2>4. Bacheca Avvisi Interni Totem</h2>
                <form action="/salva-avviso" method="post">
                    <input type="hidden" name="data_corrente" value="{str_data_lavoro}">
                    <label>Digita un messaggio urgente da far scorrere sul tabellone:</label>
                    <textarea name="testo_avviso" rows="3" placeholder="Scrivi qui gli avvisi interni...">{testo_avviso}</textarea>
                    <br><br>
                    <button type="submit" class="btn btn-success" style="width:100%;">Pubblica in Corridoio</button>
                </form>
            </div>
            
            <div class="card">
                <h2>3. Registro Sostituti del ({data_lavoro.strftime('%d/%m/%Y')})</h2>
                <table>
                    <thead><tr><th>Ora</th><th>Classe</th><th>Docente Assente</th><th>Sostituto Assegnato</th></tr></thead>
                    <tbody>{righe_tabella if righe_tabella else f"<tr><td colspan='4' style='text-align:center; color:gray;'>Nessuna variazione pianificata per questa data.</td></tr>"}</tbody>
                </table>
                <div style="margin-top:20px; display:flex; flex-direction:column; gap:12px;">
                    <a href="/totem-corridoio" target="_blank" class="btn" style="background:#ffab00; color:var(--blue-dark); text-align:center;">Apri Schermo Monitor Totem (Oggi)</a>
                    <a href="/genera-testo?data={str_data_lavoro}" target="_blank" class="btn" style="background:#008758; text-align:center; font-weight:700;">📋 GENERA LOG CIRCOLARE IN TESTO VELOCE</a>
                    
                    <div style="display:grid; grid-template-columns: 1fr 1fr; gap:10px;">
                        <a href="/esporta-log?tipo=giornaliero&data={str_data_lavoro}" class="btn btn-outline" style="text-align:center; font-size:11px; padding:10px;">Download Log Giornaliero (.csv)</a>
                        <a href="/esporta-log?tipo=mensile&data={str_data_lavoro}" class="btn btn-outline" style="text-align:center; font-size:11px; padding:10px;">Download Log Mensile (.csv)</a>
                    </div>
                    {f'<a href="/cancella-giornata?data={str_data_lavoro}" class="btn btn-danger" style="text-align:center;">Azzera Intera Giornata</a>' if sostituzioni else ''}
                </div>
            </div>
        </div>
    </body></html>
    """

# =====================================================================
# 📋 FUNZIONE DI GENERAZIONE LOG CIRCOLARE (PROTETTA)
# =====================================================================
@app.get("/genera-testo", response_class=HTMLResponse)
def genera_testo_circolare(data: str, db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    ref_date = datetime.date.fromisoformat(data)
    sostituzioni = db.query(Sostituzione).filter(Sostituzione.data == ref_date).order_by(Sostituzione.assente_id, Sostituzione.ora).all()
    
    mappa_assenti = {}
    for s in sostituzioni:
        ass = db.query(Docente).filter(Docente.id == s.assente_id).first()
        sos = db.query(Docente).filter(Docente.id == s.sostituto_id).first() if s.sostituto_id else None
        
        if ass:
            if ass.nominativo not in mappa_assenti:
                mappa_assenti[ass.nominativo] = []
            
            tipo_sost = ""
            if sos:
                if sos.tipo == "sostegno": tipo_sost = " [già compresente]"
                elif sos.tipo == "potenziamento": tipo_sost = " [da potenziamento]"
                nome_sost = f"{sos.nominativo.upper()}{tipo_sost}"
            else:
                nome_sost = "Cattedra scoperta"
                
            mappa_romani = {1: "I", 2: "II", 3: "III", 4: "IV", 5: "V", 6: "VI"}
            ora_romana = mappa_romani.get(s.ora, f"{s.ora}")
            
            mappa_assenti[ass.nominativo].append(f"{ora_romana} ora - {nome_sost} - {s.classe}")

    testo_finale = f"Buonasera colleghe e colleghi, assenze e sostituzioni di domani {ref_date.strftime('%d/%m')}.\n\n"
    
    if not mappa_assenti:
        testo_finale += "Nessuna variazione d'orario registrata. Lezioni regolari.\n"
    else:
        for docente, ore in mappa_assenti.items():
            testo_finale += f"Docente assente {docente.upper()}\n"
            for o in ore:
                testo_finale += f"{o}\n"
            testo_finale += "\n"
            
    testo_finale += "Grazie a tutte/i per la collaborazione"

    return f"""
    <!DOCTYPE html><html><head><title>Testo Circolare</title>{STILE_ITALIA}</head>
    <body style="background:var(--blue-dark); padding:30px; display:flex; justify-content:center; align-items:center; min-height:100vh; box-sizing:border-box;">
        <div class="card" style="width:100%; max-width:650px; border-top:6px solid var(--warning);">
            <h2>Copia il testo per la Bacheca / Messaggi 📋</h2>
            <p style="font-size:14px; color:gray;">Seleziona e copia tutto il testo nel riquadro qui sotto per inviarlo ai colleghi:</p>
            <textarea readonly style="width:100%; height:320px; font-family:monospace; font-size:15px; padding:15px; background:#f8f9fa; border:1px solid #ccc; border-radius:4px; resize:none;" onclick="this.select();">{testo_finale}</textarea>
            <br><br>
            <a href="/?data={data}" class="btn" style="width:100%; text-align:center; box-sizing:border-box;">⬅ Torna all'Amministrazione</a>
        </div>
    </body></html>
    """

@app.get("/elimina-voce")
def elimina_voce(id: int, data: str, db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    db.query(Sostituzione).filter(Sostituzione.id == id).delete()
    db.commit()
    return RedirectResponse(url=f"/?data={data}&msg=Sostituzione+rimossa.+Docente+riattivato.", status_code=303)

@app.post("/salva-avviso")
def salva_avviso(testo_avviso: str = Form(...), data_corrente: str = Form(...), db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    impostazione = db.query(ImpostazioniScuola).filter(ImpostazioniScuola.chiave == "avviso_totem").first()
    if not impostazione:
        impostazione = ImpostazioniScuola(chiave="avviso_totem", valore=testo_avviso.strip())
        db.add(impostazione)
    else:
        impostazione.valore = testo_avviso.strip()
    db.commit()
    return RedirectResponse(url=f"/?data={data_corrente}&msg=Avviso+pubblicato+sul+Totem!", status_code=303)

@app.get("/esporta-log")
def esporta_log(tipo: str, data: str, db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    ref_date = datetime.date.fromisoformat(data)
    if tipo == "giornaliero":
        records = db.query(Sostituzione).filter(Sostituzione.data == ref_date).order_by(Sostituzione.ora).all()
        filename = f"log_sostituzioni_{ref_date.isoformat()}.csv"
    else:
        records = db.query(Sostituzione).filter(extract('year', Sostituzione.data) == ref_date.year, extract('month', Sostituzione.data) == ref_date.month).order_by(Sostituzione.data, Sostituzione.ora).all()
        filename = f"log_sostituzioni_mensile_{ref_date.strftime('%m_%Y')}.csv"

    output = io.StringIO()
    writer = csv.writer(output, delimiter=';')
    writer.writerow(["DATA", "ORA", "CLASSE", "DOCENTE_ASSENTE", "SOSTITUTO_ASSEGNATO", "TIPOLOGIA", "ORA_REGISTRAZIONE"])
    for r in records:
        ass = db.query(Docente).filter(Docente.id == r.assente_id).first()
        sos = db.query(Docente).filter(Docente.id == r.sostituto_id).first() if r.sostituto_id else None
        writer.writerow([r.data.strftime("%d/%m/%Y"), f"{r.ora}° Ora", r.classe, ass.nominativo if ass else "SCONOSCIUTO", sos.nominativo if sos else "DA ASSEGNARE", sos.tipo if sos else "-", r.timestamp_aggiornamento])
    output.seek(0)
    return StreamingResponse(io.BytesIO(output.getvalue().encode("utf-8-sig")), media_type="text/csv", headers={"Content-Disposition": f"attachment; filename={filename}"})

@app.post("/import-motore")
async def import_motore(file: UploadFile = File(...), db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    content = await file.read()
    righe = content.decode("utf-8-sig").splitlines()
    separatore = ';' if len(righe[0].split(';')) > len(righe[0].split(',')) else ','
    reader = csv.DictReader(righe, delimiter=separatore)
    giorni = ["LUN", "MAR", "MER", "GIO", "VEN"]
    
    db.query(OrarioSettimanale).delete()
    db.query(Docente).delete()
    
    try:
        db.execute(text("DELETE FROM sqlite_sequence WHERE name='docenti';"))
        db.execute(text("DELETE FROM sqlite_sequence WHERE name='orari_completi';"))
    except Exception:
        pass
    db.commit()
    
    for row in reader:
        tipo = row.get('TIPOLOGIA', 'curriculare').strip().lower()
        nome = row.get('NOME', '').strip().upper()
        if not nome: continue
        docente = Docente(nominativo=nome, tipo=tipo)
        db.add(docente); db.flush()
        
        for g in giorni:
            for ora in range(1, 7):
                colonna = f"{g}{ora}"
                classe = row.get(colonna, '').strip() if colonna in row else ''
                if classe and classe != '':
                    db.add(OrarioSettimanale(docente_id=docente.id, giorno_settimana=g, ora_lezione=ora, classe_assegnata=classe))
    db.commit()
    return RedirectResponse(url="/?msg=File+Modello+importato.+ID+riallineati+da+1+con+successo.", status_code=303)

# =====================================================================
# ELABORA ASSENZA (MULTIORARIA DISCOBBLOCCHI, CON ESCLUSIONE FINESETTIMANA)
# =====================================================================
@app.post("/elabora-assenza")
def elabora_assenza(docente_id: str = Form(...), tipo_assenza: str = Form(...), ore_inserite: str = Form(""), data_assenza: str = Form(...), db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    if not docente_id or docente_id == "":
        return RedirectResponse(url=f"/?data={data_assenza}&msg=Errore:+Seleziona+un+docente+valido.", status_code=303)
        
    doc_id_int = int(docente_id)
    target_date = datetime.date.fromisoformat(data_assenza)
    
    if target_date.weekday() in [5, 6]:
        return RedirectResponse(url=f"/?data={data_assenza}&msg=Errore:+Sabato+e+Domenica+non+sono+giorni+di+lezione.+Seleziona+un+giorno+feriale.", status_code=303)
        
    mappa_giorni = {0: "LUN", 1: "MAR", 2: "MER", 3: "GIO", 4: "VEN"}
    giorno_settimanale_scelto = mappa_giorni[target_date.weekday()]
    slot_da_sostituire = []

    if tipo_assenza == "GIORNATA":
        orari = db.query(OrarioSettimanale).filter(OrarioSettimanale.docente_id == doc_id_int, OrarioSettimanale.giorno_settimana == giorno_settimanale_scelto).all()
        for o in orari:
            if o.classe_assegnata != 'R':
                slot_da_sostituire.append({"ora": o.ora_lezione, "classe": o.classe_assegnata})
    else:
        # 🚨 PARSING DELLE ORE MULTIPLE: smonta la stringa (es: "1,2" diventa [1, 2])
        ore_lista = []
        if ore_inserite:
            for x in ore_inserite.split(","):
                x_clean = x.strip()
                if x_clean.isdigit():
                    ore_lista.append(int(x_clean))
        
        if not ore_lista:
            return RedirectResponse(url=f"/?data={data_assenza}&msg=Errore:+Specifica+almeno+un'ora+valida+per+il+permesso.", status_code=303)

        orari_singoli = db.query(OrarioSettimanale).filter(
            OrarioSettimanale.docente_id == doc_id_int, 
            OrarioSettimanale.giorno_settimana == giorno_settimanale_scelto, 
            OrarioSettimanale.ora_lezione.in_(ore_lista)
        ).all()
        for o_sing in orari_singoli:
            if o_sing.classe_assegnata != 'R':
                slot_da_sostituire.append({"ora": o_sing.ora_lezione, "classe": o_sing.classe_assegnata})

    id_assenti_oggi = [s.assente_id for s in db.query(Sostituzione).filter(Sostituzione.data == target_date).all()]
    id_assenti_oggi.append(doc_id_int)

    ora_corrente_str = datetime.datetime.now().strftime("%H:%M:%S")
    for slot in slot_da_sostituire:
        o_target = slot["ora"]
        c_target = slot["classe"]
        sost_id = None
        
        compresenze = db.query(OrarioSettimanale).filter(
            OrarioSettimanale.giorno_settimana == giorno_settimanale_scelto, 
            OrarioSettimanale.ora_lezione == o_target, 
            OrarioSettimanale.classe_assegnata == c_target, 
            OrarioSettimanale.docente_id.not_in(id_assenti_oggi)
        ).all()
        for colla in compresenze:
            d = db.query(Docente).filter(Docente.id == colla.docente_id, Docente.tipo == "sostegno").first()
            if d: sost_id = d.id; break
            
        if not sost_id:
            disp = db.query(OrarioSettimanale).filter(
                OrarioSettimanale.giorno_settimana == giorno_settimanale_scelto, 
                OrarioSettimanale.ora_lezione == o_target, 
                OrarioSettimanale.classe_assegnata == "R", 
                OrarioSettimanale.docente_id.not_in(id_assenti_oggi)
            ).all()
            for r in disp:
                d_disp = db.query(Docente).filter(Docente.id == r.docente_id).first()
                if d_disp and d_disp.tipo != "assistente":
                    sost_id = d_disp.id; break
            
        if not sost_id:
            assistente_disp = db.query(OrarioSettimanale).filter(
                OrarioSettimanale.giorno_settimana == giorno_settimanale_scelto, 
                OrarioSettimanale.ora_lezione == o_target, 
                OrarioSettimanale.classe_assegnata == "R", 
                OrarioSettimanale.docente_id.not_in(id_assenti_oggi)
            ).all()
            for a in assistente_disp:
                d_ass = db.query(Docente).filter(Docente.id == a.docente_id, Docente.tipo == "assistente").first()
                if d_ass: sost_id = d_ass.id; break
            
        db.add(Sostituzione(data=target_date, ora=o_target, classe=c_target, assente_id=doc_id_int, sostituto_id=sost_id, timestamp_aggiornamento=ora_corrente_str))
        
    db.commit()
    return RedirectResponse(url=f"/?data={data_assenza}&msg=Provvedimenti+generati+con+successo.", status_code=303)

@app.post("/assegna-manuale")
def assegna_manuale(sostituzione_id: int = Form(...), sostituto_id: str = Form(...), data_corrente: str = Form(...), db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    sostituzione = db.query(Sostituzione).filter(Sostituzione.id == sostituzione_id).first()
    if sostituzione:
        if sostituto_id == "":
            sostituzione.sostituto_id = None
        else:
            sostituzione.sostituto_id = int(sostituto_id)
        sostituzione.timestamp_aggiornamento = datetime.datetime.now().strftime("%H:%M:%S")
        db.commit()
    return RedirectResponse(url=f"/?data={data_corrente}&msg=Sostituto+aggiornato+manualmente.", status_code=303)

@app.get("/cancella-giornata")
def cancella_giornata(data: str, db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    ref_date = datetime.date.fromisoformat(data)
    db.query(Sostituzione).filter(Sostituzione.data == ref_date).delete()
    db.commit()
    return RedirectResponse(url=f"/?data={data}&msg=Giornata+completamente+svuotata.", status_code=303)

@app.get("/reset-totale")
def reset_totale(db: Session = Depends(get_db), username: str = Depends(verifica_credenziali)):
    db.query(Sostituzione).delete()
    db.query(OrarioSettimanale).delete()
    db.query(Docente).delete()
    try:
        db.execute(text("DELETE FROM sqlite_sequence;"))
    except Exception:
        pass
    db.commit()
    return RedirectResponse(url="/?msg=Sistema+completamente+resettato.", status_code=303)

# =====================================================================
# VISTA FRONT-END: MONITOR TOTEM CAROUSEL AD ALTO IMPATTO RESPONSIVE (LIBERA)
# =====================================================================
@app.get("/totem-corridoio", response_class=HTMLResponse)
def totem_corridoio(db: Session = Depends(get_db)):
    data_oggi = datetime.date.today()
    sostituzioni = db.query(Sostituzione).filter(Sostituzione.data == data_oggi).order_by(Sostituzione.ora).all()
    
    ultimo_update_str = "Nessun inserimento"
    if sostituzioni:
        orari_modifiche = [s.timestamp_aggiornamento for s in sostituzioni if s.timestamp_aggiornamento]
        if orari_modifiche: ultimo_update_str = max(orari_modifiche)

    elementi_lista = []
    for s in sostituzioni:
        ass = db.query(Docente).filter(Docente.id == s.assente_id).first()
        sos = db.query(Docente).filter(Docente.id == s.sostituto_id).first() if s.sostituto_id else None
        
        tipo_badge = "da-assegnare"
        testo_badge = "DA ASSEGNARE"
        if s.sostituto_id and sos:
            testo_badge = sos.nominativo.upper()
            tipo_badge = "assistente" if sos.tipo == "assistente" else "ok"
            if sos.tipo == "assistente":
                testo_badge += " (ASSISTENTE)"
                
        elementi_lista.append({
            "ora": f"{s.ora}° Ora",
            "classe": s.classe,
            "assente": ass.nominativo.upper() if ass else "SCONOSCIUTO",
            "badge_testo": testo_badge,
            "badge_tipo": tipo_badge
        })

    import json
    json_sostituzioni = json.dumps(elementi_lista)

    avviso_db = db.query(ImpostazioniScuola).filter(ImpostazioniScuola.chiave == "avviso_totem").first()
    testo_bacheca = avviso_db.valore.upper() if (avviso_db and avviso_db.valore) else "NESSUN AVVISO CIRCOLARE INSERITO"
    news_string = f"★ COMUNICAZIONE INTERNA: {testo_bacheca}" + recupera_news_lato_server()

    return f"""
    <!DOCTYPE html><html>
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Sosty - Monitor Totem</title>
        {STILE_ITALIA}
        <style>
            html, body {{ height: 100%; margin: 0; padding: 0; background: #ffffff; color: var(--text-dark); display: flex; flex-direction: column; overflow: hidden; }}
            .header-totem {{ background: var(--blue-italia); color: white; padding: 15px 30px; display: flex; justify-content: space-between; align-items: center; border-bottom: 5px solid var(--blue-dark); flex-shrink: 0; }}
            .header-totem h1 {{ margin: 0; font-size: 32px; font-weight: 600; text-transform: uppercase; }}
            .header-totem p {{ margin: 0; font-size: 18px; opacity: 0.9; }}
            
            .live-clock {{ text-align: right; background: rgba(0,0,0,0.2); padding: 8px 20px; border-radius: 4px; border: 1px solid rgba(255,255,255,0.3); }}
            .live-clock .ora {{ font-size: 32px; font-weight: 600; font-variant-numeric: tabular-nums; line-height: 1; }}
            .live-clock .data {{ font-size: 17px; color: #ffab00; text-transform: uppercase; font-weight: 600; margin-top: 2px; }}
            
            .totem-container {{ flex-grow: 1; padding: 20px; display: flex; flex-direction: column; box-sizing: border-box; overflow: hidden; justify-content: flex-start; }}
            .info-bar-totem {{ background: var(--blue-light); padding: 10px 20px; border: 1px solid var(--border); border-radius: 4px; margin-bottom: 15px; font-size: 18px; color: var(--blue-dark); font-weight: 600; display: flex; justify-content: space-between; flex-shrink: 0; }}
            
            .table-wrapper {{ width: 100%; flex-grow: 1; overflow: hidden; display: flex; flex-direction: column; }}
            table.tabella-totem {{ width: 100%; border-collapse: collapse; box-shadow: 0 4px 8px rgba(0,0,0,0.04); border-radius: 4px; overflow: hidden; table-layout: fixed; }}
            table.tabella-totem th {{ background: var(--blue-dark); color: white; padding: 12px; font-size: 18px; font-weight: 600; text-transform: uppercase; text-align: left; }}
            table.tabella-totem td {{ padding: 10px 12px; font-size: 24px; font-weight: 400; border-bottom: 2px solid #dee2e6; background-color: var(--blue-light); color: var(--blue-dark); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }}
            table.tabella-totem tr:nth-child(even) td {{ background-color: #ffffff; }}
            table.tabella-totem tr:nth-child(odd) td {{ background-color: var(--blue-light); }}
            
            .badge-totem {{ padding: 6px 12px; border-radius: 4px; font-weight: 600; font-size: 24px; display: inline-block; max-width: 95%; text-overflow: ellipsis; overflow: hidden; vertical-align: middle; }}
            .badge-totem.ok {{ background: #d1e7dd; color: #0f5132; }}
            .badge-totem.da-assegnare {{ background: #f8d7da; color: #842029; border: 2px solid var(--error); }}
            .badge-totem.assistente {{ background: var(--assistente-giallo); color: var(--assistente-testo); border: 2px solid #fbc02d; }}
            
            .ticker-wrapper {{ position: fixed; bottom: 0; left: 0; width: 100%; background: var(--blue-dark); color: white; height: 50px; overflow: hidden; display: flex; align-items: center; box-shadow: 0 -2px 10px rgba(0,0,0,0.15); z-index: 9999; border-top: 4px solid var(--warning); flex-shrink: 0; }}
            .ticker-label {{ background: var(--warning); color: var(--blue-dark); padding: 0 25px; height: 100%; display: flex; align-items: center; font-weight: 600; font-size: 15px; text-transform: uppercase; white-space: nowrap; z-index: 10; box-shadow: 5px 0 10px rgba(0,0,0,0.3); }}
            .ticker-content {{ width: 100%; overflow: hidden; display: flex; align-items: center; }}
            .ticker-text-move {{ display: inline-block; white-space: nowrap; padding-left: 100%; animation: maratona 65s linear infinite; font-size: 19px; font-weight: 400; color: #ffffff; }}
            @keyframes maratona {{ 0% {{ transform: translate3d(0, 0, 0); }} 100% {{ transform: translate3d(-100%, 0, 0); }} }}
        </style>
    </head>
    <body>
        <div class="header-totem">
            <div>
                <h1>SOSTITUZIONI GIORNALIERE</h1>
                <p>I.C. TASSO LATINA | SCUOLA SECONDARIA</p>
            </div>
            <div class="live-clock">
                <div class="ora" id="live-time">00:00:00</div>
                <div class="data" id="live-date">...</div>
            </div>
        </div>
        
        <div class="totem-container">
            <div class="info-bar-totem">
                <span id="label-pagine" style="color:#e65100;">PAGINA 1 DI 1</span>
                <span>Ultimo update: {ultimo_update_str}</span>
            </div>

            <div class="table-wrapper">
                <table class="tabella-totem">
                    <thead>
                        <tr>
                            <th style="width: 15%;">ORA</th>
                            <th style="width: 15%;">CLASSE</th>
                            <th style="width: 35%;">DOCENTE ASSENTE</th>
                            <th style="width: 35%;">SOSTITUTO ASSEGNATO</th>
                        </tr>
                    </thead>
                    <tbody id="totem-table-body"></tbody>
                </table>
            </div>
        </div>

        <div class="ticker-wrapper">
            <div class="ticker-label">INFO & ULTIM'ORA</div>
            <div class="ticker-content">
                <div class="ticker-text-move">{news_string}</div>
            </div>
        </div>

        <script>
            const recordTotali = {json_sostituzioni};
            const righePerPagina = 6; 
            let paginaCorrente = 0;
            
            const pagineCarosello = [];
            for (let i = 0; i < recordTotali.length; i += righePerPagina) {{
                pagineCarosello.push(recordTotali.slice(i, i + righePerPagina));
            }}

            function muestraPaginaCarosello() {{
                const tbody = document.getElementById("totem-table-body");
                const labelPagine = document.getElementById("label-pagine");
                tbody.innerHTML = ""; 

                if (pagineCarosello.length === 0) {{
                    tbody.innerHTML = "<tr><td colspan='4' style='text-align:center; color:#7f8c8d; font-style:italic; padding: 40px;'>Nessuna variazione d'orario registrata per oggi.</td></tr>";
                    labelPagine.innerHTML = "REGISTRO VUOTO";
                    return;
                }}

                labelPagine.innerHTML = "VISUALIZZAZIONE AGGIORNATA • SCHEDA " + (paginaCorrente + 1) + " DI " + pagineCarosello.length;
                
                const datiPagina = pagineCarosello[paginaCorrente];
                datiPagina.forEach(r => {{
                    const tr = document.createElement("tr");
                    tr.className = "riga-orario";
                    tr.innerHTML = "<td>" + r.ora + "</td>" +
                                   "<td>" + r.classe + "</td>" +
                                   "<td>" + r.assente + "</td>" +
                                   "<td><span class='badge-totem " + r.badge_tipo + "'>" + r.badge_testo + "</span></td>";
                    tbody.appendChild(tr);
                }});

                if (pagineCarosello.length > 1) {{
                    paginaCorrente = (paginaCorrente + 1) % pagineCarosello.length;
                }}
            }}

            function farGirareOrologio() {{
                var d = new Date();
                var opzioniData = {{ weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }};
                document.getElementById('live-date').innerHTML = d.toLocaleDateString('it-IT', opzioniData);
                
                var hh = String(d.getHours()).padStart(2, '0');
                var mm = String(d.getMinutes()).padStart(2, '0');
                var ss = String(d.getSeconds()).padStart(2, '0');
                document.getElementById('live-time').innerHTML = hh + ":" + mm + ":" + ss;
            }}

            mostraPaginaCarosello();
            setInterval(farGirareOrologio, 1000);
            
            if (pagineCarosello.length > 1) {{
                setInterval(mostraPaginaCarosello, 15000);
            }}
            
            setTimeout(function() {{ window.location.reload(); }}, 300000);
            window.onload = farGirareOrologio;
        </script>
    </body>
    </html>
    """