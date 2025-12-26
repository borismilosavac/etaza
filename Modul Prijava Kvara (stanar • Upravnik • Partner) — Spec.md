# Zgrada Plus — Modul **Prijava kvara** (*Issue Reporting*) — kompletna specifikacija

> [!IMPORTANT]
> Jedinstveni izvor istine za **UI/UX**, **Flutter razvoj** i **operativu**. Uloge: **Stanar** (*Resident*), **Upravnik** (*Manager*), **Partner/Izvođač** (*Vendor/Contractor*). Usklađeno sa **WCAG 2.1 AA** i internim pravilima privatnosti (telefon/email opcioni).

## Sadržaj

* [0. Sažetak i ciljevi](#0-sažetak-i-ciljevi)
* [1. Domen i uloge](#1-domen-i-uloge)
* [2. Informaciona arhitektura (IA)](#2-informaciona-arhitektura-ia)
* [3. Životni ciklus i statusna mašina](#3-životni-ciklus-i-statusna-mašina)
* [4. UX tokovi po ulozi](#4-ux-tokovi-po-ulozi)
* [5. UI specifikacije i komponente](#5-ui-specifikacije-i-komponente)
* [6. Validacije, pravila i ograničenja](#6-validacije-pravila-i-ogrančenja)
* [7. Freemium vs Premium po ulozi](#7-freemium-vs-premium-po-ulozi)
* [8. Notifikacije i SLA](#8-notifikacije-i-sla)
* [9. Model podataka (Firestore referenca)](#9-model-podataka-firestore-referenca)
* [10. API ugovori (REST skica)](#10-api-ugovori-rest-skica)
* [11. Flutter implementacija — smjernice](#11-flutter-implementacija--smjernice)
* [12. Dizajn tokeni — mapiranje](#12-dizajn-tokeni--mapiranje)
* [13. Pristupačnost](#13-pristupačnost)
* [14. Analitika i KPI](#14-analitika-i-kpi)
* [15. QA scenariji](#15-qa-scenariji)
* [16. Roadmap](#16-roadmap)

---

## 0. Sažetak i ciljevi

**Svrha**: strukturisana **prijava**, **obrada**, **izvođenje** i **zatvaranje** kvarova, sa transparentnim statusima, dokazima i troškovima.
**Metrike**: prosječno vrijeme do preuzimanja (**TTA**), do početka radova (**TTS**), do zatvaranja (**TTC**), stopa SLA prekršaja.

---

## 1. Domen i uloge

* **Stanar**: prijava kvara, praćenje, poruke, potvrda rješenja i ocjena.
* **Upravnik**: prijem, triage, dodjela, planiranje, komunikacija, zatvaranje, troškovnik, izvještaji.
* **Partner/Izvođač**: preuzimanje naloga, evidencija rada, upload dokaza/računa, statusni update, završni izvještaj.

Platforme: Stanar (mobilna + web), Upravnik (web + mobilna), Partner (mobilna + web light).

---

## 2. Informaciona arhitektura (IA)

```
Prijava kvara
 ├─ Stanar
 │   ├─ Lista mojih prijava
 │   ├─ Novi kvar (wizard 5 koraka)
 │   └─ Detalj prijave (timeline, poruke, ocjena)
 ├─ Upravnik
 │   ├─ Inbox / Kanban po statusima
 │   ├─ Detalj prijave (master–detail)
 │   ├─ Dodjela i planiranje (izvođač, termin, SLA)
 │   ├─ Troškovnik i dokumenti
 │   └─ Izvještaji
 └─ Partner/Izvođač
     ├─ Moji nalozi (današnji / aktivni / završeni)
     ├─ Detalj naloga (zadaci, materijal, foto‑dokazi)
     └─ Predaj završni izvještaj
```

---

## 3. Životni ciklus i statusna mašina

```
Primljeno → U obradi → U radu → Čeka potvrdu → Zatvoreno
        ↘ Odbijeno
         ↘ Reotvoreno (iz „Čeka potvrdu“ ako stanar odbije)
```

**SLA tajmeri**:

* `SLA.triage` (Primljeno→U obradi),
* `SLA.start` (U obradi→U radu),
* `SLA.finish` (U radu→Čeka potvrdu).

**Događaji**: kreirano, dodijeljeno, promjena prioriteta, upload dokaza, trošak dodat, izvještaj partnera, zatvoreno, reotvoreno.

---

## 4. UX tokovi po ulozi

### 4.1 Stanar

1. **Novi kvar (wizard)**:

   * **Opis** (naslov, detalji) →
   * **Dokazi** (foto/video/audio) →
   * **Lokacija** (moj stan / zajednički prostor: ulaz/sprat/prostorija) i **Kategorija** (voda, struja, lift, čistoća, ostalo) →
   * **Prioritet** (normalno/hitno) i **Vidljivost** (privatno/javno) →
   * **Pregled i slanje**.
2. **Detalj prijave**: header (status, prioritet, kategorija), opis, prilozi, timeline, poruke. Akcije: dodaj poruku/dokaz, povuci prijavu (samo u **Primljeno**), nakon zatvaranja unesi ocjenu/komentar.
3. **Notifikacije**: promjene statusa, traženje dodatnih informacija, poziv na ocjenu.

### 4.2 Upravnik

1. **Inbox/Kanban**: filtrira po statusu, kategoriji, prioritetu, zgradi/ulazu, izvođaču, SLA riziku; drag‑and‑drop po kolonama (web).
2. **Triage**: validacija opisa/dokaza, korekcija kategorije/prioriteta, dopunska pitanja stanaru.
3. **Dodjela**: interni tehničar ili partner; definisanje roka/SLA i termina.
4. **Planiranje/Izvođenje**: praćenje statusa „U radu“, komunikacija, dodavanje troškova i dokumenata.
5. **Zatvaranje**: verifikacija izvještaja, stvarni trošak, promjena statusa u „Čeka potvrdu“, po potvrdi stanara — „Zatvoreno“.

### 4.3 Partner/Izvođač

1. **Moji nalozi**: Today/Active/All; sort po terminu/SLA.
2. **Detalj**: opis kvara, kontakt‑meta (bez PII osim ako eksplicitno dozvoljeno), lokacija (mapa/opis), traženi termin.
3. **Radne akcije**: start job, upload foto/video, dodaj materijal i sate, ostavi napomenu, predaj završni izvještaj.

---

## 5. UI specifikacije i komponente

### 5.1 Stanar — ekrani

* **Lista**: kartice sa naslovom, statusom (badge), datumom i prioritetom. Filteri: Status, Kategorija, Period.
* **Novi kvar (wizard)**: 5 koraka, validacije po koraku, autosave draft. Upload do 3 fajla (freemium) / 5 (premium).
* **Detalj**: timeline promjena (neizbrisiv), thread 1:1 sa upravnikom, ocjena (1–5) po zatvaranju.

### 5.2 Upravnik — ekrani

* **Inbox/Kanban**: kolone po statusu; kartica s ključnim meta (kategorija, prioritet, SLA indikator). Bulk: označi pročitano/riješeno, dodijeli izvođača.
* **Detalj**: master–detail; sekcije: opis, prilozi, lokacija, kategorija, troškovnik, dokumenti, timeline, poruke. Akcije: dodijeli, zakazivanje, promijeni status, kreiraj nalog partneru, dodaj trošak/dokument, zatvori/odbij.

### 5.3 Partner — ekrani

* **Moji nalozi**: lista + map view (opciono).
* **Detalj naloga**: Start/Pause/Finish, upload dokaza, uneseni sati i materijal, generiši završni izvještaj PDF.

### 5.4 Komponente (AI‑readable props)

* `IssueCard { title, status, priority, category, createdAt, hasMedia }`
* `Wizard.Step { id, title, valid, onNext }`
* `AttachmentUploader { accept:[jpg,png,mp4,pdf], maxCount:3|5, maxSizeMB:20|50 }`
* `StatusBadge { state:"primljeno|u_obradi|u_radu|ceka_potvrdu|zatvoreno|odbijeno|reotvoreno" }`
* `SlaChip { risk:"ok|warning|breach", dueAt }`
* `WorkOrder { contractorId, scheduledFor, eta, sla }`

---

## 6. Validacije, pravila i ograničenja

* **Naslov**: 5–120; **Opis**: ≥ 10 znakova.
* **Prilozi**: do 3 (freemium) / 5 (premium); ≤ 20 MB po fajlu (freemium) / 50 MB (premium); tipovi: jpg/png/mp4/pdf.
* **Lokacija**: obavezna; ako „zajednički prostor“ — obavezno izabrati ulaz/sprat/prostoriju.
* **Prioritet „hitno“**: obavezno kratko obrazloženje ili izbor simptoma (npr. aktivno curenje).
* **Povlačenje prijave**: dozvoljeno samo u statusu **Primljeno**.
* **Troškovnik/Dokumenti**: samo premium upravnik/partner.
* **Brisanje poruka**: nema bulk; samo pojedinačno sa audit logom (privilege: upravnik).

---

## 7. Freemium vs Premium po ulozi

### 7.1 Stanar

| Kategorija | **Freemium**             | **Premium**                              |
| ---------- | ------------------------ | ---------------------------------------- |
| Wizard     | 5 koraka, osnovna polja  | Isto                                     |
| Prilozi    | 3×20 MB                  | 5×50 MB, brži upload                     |
| Prioritet  | Normalno/Hitno           | Hitno + push podsjetnik ako miruje >48 h |
| Poruke     | 1:1 thread sa upravnikom | Isto                                     |
| Uređivanje | Do statusa **U obradi**  | Do **U radu**                            |
| Ocjena     | 1–5 zvjezdica            | Ocjena + komentar, historija 12 mj.      |
| Arhiva     | 6 mj.                    | 12 mj. + izvoz PDF                       |
| Offline    | Draft + zadnjih 10       | Draft + zadnjih 15                       |

### 7.2 Upravnik

| Kategorija   | **Freemium**                | **Premium**                                     |
| ------------ | --------------------------- | ----------------------------------------------- |
| Inbox        | Lista + osnovni filteri     | + **Kanban**, **SLA** indikator, sort po riziku |
| Dodjela      | Ručna (interno)             | + Partner lista, auto obavještenje, kalendar    |
| Planiranje   | Rok, napomena               | SLA rok, podsjetnici, rasporedi                 |
| Troškovnik   | —                           | Stavke troška, valuta, dokumenti/računi         |
| Dokumenti    | Upload u detalju            | Faza‑po‑faza, generisani **PDF** izvještaj      |
| Radni nalozi | —                           | Kreiraj/pošalji partneru, status „U toku“       |
| Izvještaji   | Osnovne brojke              | Prosj. vrijeme, SLA prekršaji, CSV/XLS          |
| Notifikacije | Push na novu prijavu/poruku | + SMS za „Hitno“, SLA breach alert              |
| Arhiva       | 12 mj.                      | 24 mj. + PDF izvoz                              |

### 7.3 Partner/Izvođač

| Kategorija      | **Freemium**       | **Premium**                          |
| --------------- | ------------------ | ------------------------------------ |
| Prijem naloga   | Web light          | Mobilna + web, geolokacija (opciono) |
| Evidencija rada | Start/Finish, foto | + Pauze, sati, materijal             |
| Dokumenti       | —                  | Upload računa, generiši završni PDF  |
| Notifikacije    | In‑app             | + Push/SMS (opt‑in)                  |

---

## 8. Notifikacije i SLA

* **Stanar**: potvrda prijave; promjene statusa; traženje dodatnih informacija; poziv na ocjenu.
* **Upravnik**: nova prijava; „hitno“; prva poruka stanara; SLA rizik (žuti), SLA breach (crveni).
* **Partner**: novi nalog; promjena termina; podsjetnik pred rok; odbijanje/vraćanje.

**SLA pravila (premium)**: definisati per‑kategorija/per‑zgrada; default: `triage 4h`, `start 24h`, `finish 72h`.

---

## 9. Model podataka (Firestore referenca)

```json
/issues/{id} : {
  "title": "string",
  "description": "string",
  "reporterId": "uid",
  "buildingId": "BLDG-1",
  "entrance": "A",
  "floor": 2,
  "unitId": "A-202",
  "commonArea": true,
  "category": "voda|struja|lift|čistoća|ostalo",
  "priority": "normalno|hitno",
  "status": "primljeno|u_obradi|u_radu|ceka_potvrdu|zatvoreno|odbijeno|reotvoreno",
  "attachments": [{"name":"file.jpg","url":"…","type":"image/jpeg","size":123456}],
  "assignee": {"type":"internal|partner","id":"cid","name":"Contractor d.o.o."},
  "sla": {"triageDueAt": 1735200000000, "startDueAt": 1735286400000, "finishDueAt": 1735459200000, "risk":"ok|warning|breach"},
  "costs": [{"label":"Materijal","amount":120.00,"currency":"BAM","docUrl":"…"}],
  "timeline": [{"at":1735200300000,"by":"uid","action":"status_change","note":"U rad"}],
  "createdAt": 1735190000000,
  "updatedAt": 1735200300000,
  "closedAt": null
}

/issues/{id}/messages/{msgId} : {
  "from": "stanar|upravnik|izvođač",
  "body": "string",
  "attachments": [{"name":"clip.mp4","url":"…","type":"video/mp4","size":10485760}],
  "createdAt": 1735200400000,
  "readBy": ["uid1","uid2"]
}

/workOrders/{id} : {
  "issueId": "…",
  "contractorId": "…",
  "scheduledFor": 1735286400000,
  "status": "kreirano|poslato|prihvaćeno|u_toku|predato|odbijeno",
  "report": {"text":"…","attachments":[…]}
}
```

**Indeksi**: `(buildingId, status)`, `(status, priority, createdAt DESC)`, `(assignee.id, status)`.

---

## 10. API ugovori (REST skica)

* `POST /issues` — kreiraj prijavu.
* `GET /issues` — lista (filteri: status, kategorija, prioritet, period, zgrada, assignee).
* `GET /issues/{id}` — detalj.
* `PATCH /issues/{id}` — promjena polja/statusa.
* `POST /issues/{id}/messages` — poruka.
* `POST /issues/{id}/assign` — dodjela (internal/partner).
* `POST /issues/{id}/costs` — dodaj trošak (premium).
* `POST /workorders` — kreiraj nalog partneru (premium).
* `PATCH /workorders/{id}` — status/izvještaj.

HTTP: `400 VAL_*`, `401 UNAUTH`, `403 AUTH_PLAN_REQUIRED`, `404 NF`, `409 CONFLICT`, `413 PAYLOAD_TOO_LARGE`.

---

## 11. Flutter implementacija — smjernice

* **Arhitektura**: data→domain→presentation; **Riverpod**/**Bloc**; **go_router**; offline keš sa **Hive/Drift** (LRU 10/15).
* **Rute**: `/issues`, `/issues/new`, `/issues/:id`, `/workorders/:id`.
* **Stanja (sealed)**: `IssueFeedState`, `IssueDetailState`, `WorkOrderState`, `ThreadState` (new/sending/open/offlineQueue/fail).
* **Uploader**: `image_picker`/`file_picker`, chunked upload, progress bar; tipovi `jpg/png/mp4/pdf`.
* **Mapiranje tokens**: primarna boja za CTA, status badge po statusu i SLA rizičnosti (success/warning/error tints).
* **PWA/offline**: draft wizard, retry poruka/upload, read‑only kad je offline.
* **Pristupačnost**: `Semantics`, `MediaQuery.textScaleFactor`, fokus outline 2 px, kontrast AA.

---

## 12. Dizajn tokeni — mapiranje

* **Button.Primary**: bg `blue.600` → hover `blue.700`, text `base.white`, focusRing `blue.300`, radius `12`.
* **Badge.Status**: `success/warning/error` varijante; pozadine `100`, tekst `700`, border `200`.
* **Card**: bg `gray modern.50`, border `200`, radius `16`, shadow `sm`.
* **Inputs**: border `gray modern.300`, focus `blue.300`.

> Konkretne vrijednosti preuzeti iz `design-tokens.tokens.json` i mapirati u Figma/Flutter ThemeData.

---

## 13. Pristupačnost

* Kontrast ≥ 4.5:1, tap target ≥ 44 px, label/aria za sve kontrole.
* Tastatura: tab redoslijed, `Enter` potvrda, `Esc` zatvaranje.
* Tekst skalira po OS; naslovi 1.25×, tijelo 1×, meta 0.875×.

---

## 14. Analitika i KPI

**Eventi**: `issue_create`, `issue_assign`, `issue_status_change`, `issue_message_sent`, `workorder_create`, `workorder_submit_report`, `sla_breach`.
**KPI**: TTA/TTS/TTC, % SLA breach, # re‑open, prosj. ocjena, % prijava sa potpunim dokazima.

---

## 15. QA scenariji

* Kreiranje prijave sa 3 priloga, offline draft → online submit.
* Dodjela partneru, kalendarski termin, push partneru, početak rada, upload dokaza, završni izvještaj, zatvaranje.
* SLA: simulirati rizik i breach; upozorenja upravniku.
* Re‑open: stanar odbija rješenje → vraća se u „U radu“.
* Troškovnik: dodavanje stavke, upload računa, generisani PDF izvještaj (premium).

---

## 16. Roadmap

* **v1.1**: Napredne automatizacije (auto‑assign po kategoriji, radno vrijeme), kalendari izvođača.
* **v1.2**: Integracije (Helpdesk/CRM, gradski API), QR nalog za zajedničke prostore.
* **v1.3**: Geofencing dolaska partnera, e‑potpis predaje.

— Kraj specifikacije —
