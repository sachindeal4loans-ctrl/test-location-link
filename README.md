import os, sqlite3
from datetime import datetime, timezone
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import HTMLResponse
from pydantic import BaseModel, Field

DB=os.getenv("LOCATION_DB","locations.db")
TOKEN=os.getenv("ADMIN_TOKEN","")
app=FastAPI()
def init():
    with sqlite3.connect(DB) as c:
        c.execute("CREATE TABLE IF NOT EXISTS locations(id INTEGER PRIMARY KEY, created_at TEXT, latitude REAL, longitude REAL, accuracy REAL)")
init()
class Submission(BaseModel):
    latitude: float=Field(ge=-90,le=90)
    longitude: float=Field(ge=-180,le=180)
    accuracy: float|None=None

@app.get("/",response_class=HTMLResponse)
def home():
 return """<!doctype html><html><meta charset=utf-8><meta name=viewport content='width=device-width,initial-scale=1'><title>Share location</title>
 <style>body{font:16px system-ui;background:#f3f6fb;color:#172033;margin:0}main{max-width:560px;margin:8vh auto;background:white;padding:28px;border-radius:16px}button{padding:12px 18px;background:#175cd3;color:white;border:0;border-radius:9px;font-size:1rem}p{line-height:1.5}</style>
 <main><h1>Share your location</h1><p>If you continue, your current location will be sent to the person operating this page. Press the button and approve your browser's location permission to proceed. You can decline and leave without sharing.</p><button id=b>Share my location</button><p id=s role=status></p></main>
 <script>const b=document.getElementById('b'),s=document.getElementById('s');b.onclick=()=>{if(!navigator.geolocation){s.textContent='Location is not supported.';return}b.disabled=true;s.textContent='Requesting permission…';navigator.geolocation.getCurrentPosition(async p=>{if(!confirm('Send your location to the page operator?')){s.textContent='Location not sent.';b.disabled=false;return}try{let r=await fetch('/api/location',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({latitude:p.coords.latitude,longitude:p.coords.longitude,accuracy:p.coords.accuracy})});if(!r.ok)throw 0;s.textContent='Location shared successfully.'}catch(e){s.textContent='Could not send location. Try again later.';b.disabled=false}},e=>{s.textContent=e.code===1?'Permission denied; nothing was shared.':'Could not get location.';b.disabled=false},{enableHighAccuracy:true,timeout:15000,maximumAge:0})}</script></html>"""

@app.post("/api/location")
def save(x:Submission):
 with sqlite3.connect(DB) as c:
  c.execute("INSERT INTO locations(created_at,latitude,longitude,accuracy) VALUES(?,?,?,?)",(datetime.now(timezone.utc).isoformat(),x.latitude,x.longitude,x.accuracy))
 return {"ok":True}

@app.get("/admin",response_class=HTMLResponse)
def admin():
 return """<!doctype html><meta charset=utf-8><meta name=viewport content='width=device-width,initial-scale=1'><title>Shared locations</title><h1>Shared locations</h1><p>Enter the admin token configured by the operator.</p><input id=t type=password placeholder='Admin token'><button id=b>Load</button><div id=r></div><script>b.onclick=async()=>{r.textContent='Loading…';try{let x=await fetch('/api/admin/locations',{headers:{'X-Admin-Token':t.value}});if(!x.ok)throw 0;let a=await x.json();r.innerHTML=a.length?'<table border=1 cellpadding=8><tr><th>Time (UTC)</th><th>Latitude</th><th>Longitude</th><th>Map</th></tr>'+a.map(v=>'<tr><td>'+v.created_at+'</td><td>'+v.latitude+'</td><td>'+v.longitude+'</td><td><a target=_blank rel=noopener href="https://maps.google.com/?q='+v.latitude+','+v.longitude+'">Open map</a></td></tr>').join('')+'</table>':'No shared locations yet'}catch(e){r.textContent='Access denied or server error'}}</script>"""

@app.get("/api/admin/locations")
def locations(req:Request):
 if not TOKEN or req.headers.get("X-Admin-Token")!=TOKEN: raise HTTPException(401,"Unauthorized")
 with sqlite3.connect(DB) as c:
  c.row_factory=sqlite3.Row
  return [dict(x) for x in c.execute("SELECT created_at,latitude,longitude,accuracy FROM locations ORDER BY id DESC LIMIT 200")]
