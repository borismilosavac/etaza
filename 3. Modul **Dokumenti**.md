# Modul **Dokumenti/Resursi** (*Documents/Resources*) — kompletna specifikacija v1.0

> [!IMPORTANT]
> **Svrha** (*Purpose*): Centralni registar dokumenata i fajlova za zgradu, sa kontrolom vidljivosti, verzijama, rokovima, audit tragom i vezama ka izvorima (npr. **Obavještenja** (*Notifications*), **Prijava kvara** (*Issue reporting*), **Finansije** (*Finance*)). Dokument je GitHub-friendly (Markdown), spreman za AI dizajn alate, UI/UX i Flutter implementaciju.

## Sadržaj

* [0. Princip i scope](#0-princip-i-scope)
* [1. Uloge i prava pristupa](#1-uloge-i-prava-pristupa)
* [2. Kategorije dokumenata](#2-kategorije-dokumenata)
* [3. Vidljivost i privatnost](#3-vidljivost-i-privatnost)
* [4. Informaciona arhitektura (IA)](#4-informaciona-arhitektura-ia)
* [5. Sortiranje kartica (algoritam)](#5-sortiranje-kartica-algoritam)
* [6. UX tokovi po ulozi](#6-ux-tokovi-po-ulozi)
* [7. UI specifikacije i komponente](#7-ui-specifikacije-i-komponente)
* [8. Validacije i poslovna pravila](#8-validacije-i-poslovna-pravila)
* [9. Freemium vs Premium (po ulozi)](#9-freemium-vs-premium-po-ulozi)
* [10. Model podataka (Firestore referenca)](#10-model-podataka-firestore-referenca)
* [11. API ugovori (REST skica)](#11-api-ugovori-rest-skica)
* [12. Flutter implementacija — smjernice](#12-flutter-implementacija--smjernice)
* [13. Dizajn tokeni — mapiranje](#13-dizajn-tokeni--mapiranje)
* [14. Pristupačnost](#14-pristupačnost)
* [15. Analitika i KPI](#15-analitika-i-kpi)
* [16. QA scenariji](#16-qa-scenariji)

---

## 0. Princip i scope

### 0.1 Centralni registar

**Dokumenti/Resursi** (*Documents/Resources*) su **jedino mjesto** gdje se fajlovi:

* klasifikuju,
* verzionišu,
* indeksiraju (pretraga),
* kontrolišu po vidljivosti,
* čuvaju uz audit.

### 0.2 Uključivanje fajlova iz drugih modula

Svi upload fajlovi iz:

* **Obavještenja** (*Notifications*) — prilozi obavještenja + prilozi u thread porukama,
* **Prijava kvara** (*Issue reporting*) — dokazi prije/poslije, izvještaji izvođača, računi/troškovi,
* **Finansije** (*Finance*) — računi, predračuni, ponude, ugovori,

ulaze u **registar**, ali sa metapodacima: **sourceModule**, **sourceType**, **sourceId**, **visibilityScope**.

### 0.3 “Registracija” vs “objava u biblioteci”

* Fajl se uvijek registruje u sistem.
* U javnu biblioteku zgrade (`building` scope) ulazi samo ako:

  * je inicijalno kreiran kao javni resurs, ili
  * upravnik eksplicitno uradi akciju: **Objavi u resurse zgrade** (*Publish to building resources*).

---

## 1. Uloge i prava pristupa

### 1.1 Uloge

* **Stanar** (*Resident*)
* **Upravnik** (*Manager*)
* **Partner/Izvođač** (*Vendor/Contractor*)

### 1.2 RBAC matrica

| Akcija                                  |                Stanar |                Upravnik |               Partner |
| --------------------------------------- | --------------------: | ----------------------: | --------------------: |
| Pregled javnih resursa zgrade           |                     ✓ |                       ✓ | ✓ (ako je relevantno) |
| Pregled segmentnih resursa              | ✓ (ako je u segmentu) |                       ✓ | ✓ (ako je relevantno) |
| Pregled privatnih priloga (thread/kvar) |        ✓ (samo svoje) |                       ✓ | ✓ (samo svoje naloge) |
| Upload novog javnog resursa             |                     – |                       ✓ |   ✓ (samo kroz nalog) |
| Upload privatnog fajla                  |  ✓ (u svom kontekstu) |                       ✓ |     ✓ (u svom nalogu) |
| Objavi u resurse zgrade                 |                     – |                       ✓ |                     – |
| Edit metapodataka (naziv, tip, tagovi)  |                     – |                       ✓ |        ✓ (samo svoje) |
| Verzionisanje (nova verzija/restore)    |                     – |                       ✓ |                     – |
| Brisanje fajla                          |                     – | ✓ (soft delete + audit) |                     – |
| Export (PDF/ZIP)                        |                     – |                       ✓ |        ✓ (samo svoje) |

---

## 2. Kategorije dokumenata

### 2.1 Tri biblioteke (UI sekcije)

1. **Bazični resursi zgrade** (*Building library*)

* pravilnici, kućni red, zapisnici, ugovori, obrasci, uputstva

2. **Operativni prilozi** (*Operational attachments*)

* fajlovi nastali iz obavještenja i kvarova (dokazi, fotografije, uputstva)

3. **Finansijski dokumenti** (*Financial docs*)

* računi, ponude, predračuni, garancije, otpremnice, obračuni

### 2.2 Tipovi (documentType)

`policy | minutes | contract | form | manual | photo | video | report | invoice | quote | warranty | other`

### 2.3 Izvori (sourceModule/sourceType)

* `resources/standalone`
* `notifications/notice_attachment`
* `notifications/notice_thread_attachment`
* `issues/issue_evidence`
* `issues/contractor_report`
* `issues/cost_doc`
* `finance/invoice_doc`

---

## 3. Vidljivost i privatnost

### 3.1 Vidljivost (visibilityScope)

* `building` — svi stanari zgrade
* `segment` — ulaz/sprat/grupa
* `unit` — određeni stan
* `private` — privatni thread ili interni upravnik↔partner

### 3.2 Pravilo protiv curenja podataka

Prilog iz `private` konteksta **ne smije** postati `building` bez eksplicitne akcije upravnika:

* **Objavi u resurse zgrade** (*Publish to building resources*)
* opciono: redakcija prije objave (uklanjanje PII)

### 3.3 Telefon/email opcioni

U karticama i listama resursa **ne prikazivati** telefon/email. Identitet autora:

* stanar: `unitLabel` (npr. “B-204”) + ime samo ako je već dostupno u profilu.

---

## 4. Informaciona arhitektura (IA)

```
Dokumenti/Resursi
 ├─ Lista (grid/list)
 │   ├─ Sekcije: Bazični | Operativni | Finansijski
 │   ├─ Filteri: tip, status, rok, izvor, vidljivost, vlasnik, tagovi
 │   ├─ Pretraga: naziv, tagovi, (OCR premium)
 │   └─ Bulk akcije (upravnik): arhiviraj, promijeni vidljivost, export
 ├─ Upload / Novi dokument
 │   ├─ Fajl + naziv
 │   ├─ Tip + tagovi
 │   ├─ Vidljivost + segment
 │   ├─ Rok/važi do (opciono)
 │   └─ Pregled & objava
 └─ Detalj dokumenta
     ├─ Preview / Download
     ├─ Metapodaci + izvor (Vidi izvor)
     ├─ Verzije (upravnik)
     ├─ Audit log (upravnik)
     └─ Akcije: Objavi, Arhiviraj, Export, Zamijeni verzijom
```

---

## 5. Sortiranje kartica (algoritam)

### 5.1 Osnovno pravilo

* **Primarni ključ**: prioritet prikaza (pin/obavezno/rok/novost/status)
* **Sekundarni ključ**: relevantni datum
* **Tie-breaker**: `id DESC`

### 5.2 Stanar — default sort

Slojevi:

1. **Pinovano** (*Pinned*)
2. **Obavezno** (*Required*)
3. **Sa rokom** (*Due*) — `dueAt ASC`
4. **Novo/izmijenjeno** (*New/Updated*) — `updatedAt DESC`
5. **Ostalo aktivno** (*Active*) — `updatedAt DESC`
6. **Arhiva** (*Archived*) — samo uz filter, `archivedAt DESC`
7. **Zamijenjeno** (*Superseded*) — samo uz filter, `supersededAt DESC`

### 5.3 Upravnik — default sort

Slojevi:

1. **Pinovano**
2. **Čeka radnju** (*Action needed*)
3. **Sa rokom** — `dueAt ASC`
4. **Draft** — `updatedAt DESC`
5. **Aktivno** — `updatedAt DESC`
6. **Zamijenjeno** — `supersededAt DESC`
7. **Arhiva** — `archivedAt DESC`

### 5.4 Partner — default sort

1. **Sa rokom** — `dueAt ASC`
2. **Novi zahtjevi** — `createdAt DESC`
3. **Aktivno** — `updatedAt DESC`
4. **Finansije** — `issuedAt DESC`
5. **Arhiva** — uz filter

---

## 6. UX tokovi po ulozi

### 6.1 Stanar

1. Otvara modul → vidi bazične i segmentne resurse.
2. Pretraga/filter → otvara detalj.
3. Download/preview.
4. (Opc.) potvrda čitanja (*Acknowledge*) ako je dokument označen kao **Obavezno**.

### 6.2 Upravnik

1. Upload novog dokumenta → klasifikacija → scope/segment → objava.
2. Iz drugih modula: akcija **Sačuvaj u resurse** (*Save to resources*) → zatim po potrebi **Objavi**.
3. Nova verzija dokumenta: upload → automatski supersede prethodnu.
4. Bulk: arhiviraj, promijeni scope, export.

### 6.3 Partner/Izvođač

1. U okviru naloga upload izvještaj/račun/dokaz.
2. Edit metapodataka samo za svoje fajlove.
3. Nema publish u building scope.

---

## 7. UI specifikacije i komponente

### 7.1 Lista (mobilno/web)

Kartica dokumenta prikazuje:

* naziv (2 reda)
* tip badge
* scope chip (`building/segment/private`)
* meta: izvor (ikonica + naziv modula)
* datum (`updatedAt` ili `dueAt`)
* indikatori: `pinned`, `required`, `dueSoon`, `new`

### 7.2 Detalj dokumenta

Sekcije:

* preview (premium inline) / download
* metapodaci: tip, tagovi, veličina, vlasnik, scope
* izvor: **Vidi izvor** (*View source*) deep link
* verzije (upravnik)
* audit (upravnik)

### 7.3 Upload (upravnik/partner)

Koraci:

1. fajl + naziv
2. tip + tagovi
3. scope + segment
4. rok/važi do
5. pregled i objava

### 7.4 Komponente (AI-readable props)

* `DocCard { title, docType, scope, sourceModule, updatedAt, dueAt?, flags:[pinned,required,new,dueSoon] }`
* `DocFilterBar { search, type[], status[], scope[], source[], due[] }`
* `DocUploader { accept:[pdf,jpg,png,mp4,docx,xlsx], maxCount, maxSizeMB }`
* `DocPreview { mode: inline|downloadOnly }`
* `VersionList { items[], canRestore }`
* `AuditLog { entries[] }`

---

## 8. Validacije i poslovna pravila

### 8.1 Validacije

* naziv: 3–160
* tipovi: `pdf, jpg, png, mp4, docx, xlsx`
* max size: freemium 20 MB / premium 50 MB
* max count per upload: 5 (upravnik), 3 (partner)

### 8.2 Verzionisanje

* nova verzija postaje **aktivna**
* prethodna dobija status **Zamijenjeno** (*Superseded*)
* restore samo upravnik, obavezno u audit

### 8.3 Objavljivanje privatnih priloga

Publish zahtijeva:

* potvrdu scope-a
* opciju redakcije PII
* audit zapis

---

## 9. Freemium vs Premium (po ulozi)

### 9.1 Stanar

| Funkcija                 | Freemium | Premium |
| ------------------------ | -------- | ------- |
| Pregled resursa          | ✓        | ✓       |
| Filteri (osnovni)        | ✓        | ✓       |
| Pretraga po nazivu       | ✓        | ✓       |
| Inline preview PDF/image | –        | ✓       |
| OCR pretraga sadržaja    | –        | ✓       |
| Arhiva (12 mj.)          | –        | ✓       |
| Offline keš              | 10       | 20      |

### 9.2 Upravnik

| Funkcija                   | Freemium | Premium |
| -------------------------- | -------- | ------- |
| Upload + klasifikacija     | ✓        | ✓       |
| Segment scope              | ✓        | ✓       |
| Verzionisanje              | ✓        | ✓       |
| Inline preview + anotacije | –        | ✓       |
| OCR indeksiranje           | –        | ✓       |
| Bulk export ZIP/PDF        | ✓        | ✓       |
| Duži audit retention       | –        | ✓       |
| Automatika publish pravila | –        | ✓       |

### 9.3 Partner

| Funkcija                  | Freemium | Premium |
| ------------------------- | -------- | ------- |
| Upload kroz nalog         | ✓        | ✓       |
| Pregled svojih dokumenata | ✓        | ✓       |
| Generisani završni PDF    | –        | ✓       |
| OCR / indeksiranje        | –        | ✓       |

---

## 10. Model podataka (Firestore referenca)

```json
/resources/{docId} : {
  "title": "string",
  "docType": "policy|report|invoice|photo|video|form|other",
  "status": "active|draft|archived|superseded",
  "scope": "building|segment|unit|private",
  "segment": {"entrance":"A","floor":2,"groupId":null},
  "ownerRole": "manager|resident|contractor",
  "ownerId": "uid|cid",
  "sourceModule": "resources|notifications|issues|finance",
  "sourceType": "standalone|notice_attachment|notice_thread_attachment|issue_evidence|cost_doc|contractor_report|invoice_doc",
  "sourceId": "abc123",
  "file": {"name":"x.pdf","url":"...","mime":"application/pdf","size":123456},
  "tags": ["string"],
  "dueAt": 1735286400000,
  "validUntil": 1737696000000,
  "versionGroupId": "vg-1",
  "version": 3,
  "createdAt": 1735190000000,
  "updatedAt": 1735200300000,
  "archivedAt": null,
  "supersededAt": null,
  "flags": {"pinned": false, "required": false}
}

/resources/{docId}/versions/{verId} : {
  "version": 2,
  "file": {"name":"x-v2.pdf","url":"..."},
  "createdAt": 1734000000000
}

/resources/{docId}/audit/{auditId} : {
  "at": 1735200400000,
  "by": "uid",
  "action": "create|update_meta|publish|change_scope|archive|restore_version|soft_delete",
  "note": "string"
}
```

Indeksi: `(scope, updatedAt DESC)`, `(docType, updatedAt DESC)`, `(sourceModule, sourceId)`.

---

## 11. API ugovori (REST skica)

* `POST /resources` — kreiraj (upload)
* `GET /resources` — lista (filteri: type/status/scope/source/due/tags)
* `GET /resources/{id}` — detalj
* `PATCH /resources/{id}` — meta/scope/status
* `POST /resources/{id}/publish` — publish (private→building/segment)
* `POST /resources/{id}/versions` — nova verzija
* `POST /resources/{id}/export` — export PDF/ZIP
* `GET /resources/{id}/audit` — audit (upravnik)

Greške: `400 VAL_*`, `403 AUTH_PLAN_REQUIRED`, `409 CONFLICT`, `413 PAYLOAD_TOO_LARGE`.

---

## 12. Flutter implementacija — smjernice

* **Arhitektura**: `data → domain → presentation`
* **State**: Riverpod ili Bloc (jedno)
* **Rute**: `/resources`, `/resources/new`, `/resources/:id`
* **Offline**: Hive/Drift, LRU 10/20; keš metapodataka
* **Preview**: premium inline (PDF viewer / image), freemium download-only
* **Uploader**: progress, retry per fajl, resumable upload (ako backend podržava)
* **Entitlements**: provjera na API sloju; UI reflektuje mogućnosti

---

## 13. Dizajn tokeni — mapiranje

* `Button.Primary`: bg `blue.600`, hover `blue.700`, focusRing `blue.300`, radius `12`
* `Card`: bg `gray.50`, border `gray.200`, radius `16`
* `Badge.DueSoon`: bg `warning.100`, text `warning.700`, border `warning.200`
* `Badge.Archived`: bg `gray.100`, text `gray.700`
* `Focus`: 2 px outline (WCAG AA)

Vrijednosti preuzeti iz `design-tokens.tokens.json` i mapirati u Figma varijable + Flutter `ThemeData`.

---

## 14. Pristupačnost

* Kontrast ≥ 4.5:1; tap target ≥ 44 px
* `aria-live=polite` / `Semantics` za promjene
* Fokus ostaje na istoj kartici po `id` nakon refresh-a
* Sort indikator: ikona + tekst (ne samo boja)

---

## 15. Analitika i KPI

Eventi:

* `resource_open`, `resource_download`, `resource_search`, `resource_filter`
* `resource_upload`, `resource_publish`, `resource_version_add`, `resource_export`

KPI:

* % dokumenata sa tipom i tagovima
* prosj. vrijeme search→open
* dueSoon→download konverzija
* broj publish akcija iz privatnih priloga

---

## 16. QA scenariji

* Privatni prilog iz thread-a → nije vidljiv stanarima dok upravnik ne publish
* Verzionisanje: nova verzija aktivna, stara u “Verzije”
* Sort: `dueAt ASC` u due sloju, stabilan tie-breaker
* Premium: inline preview + OCR radi; freemium: download-only
* Offline: lista i detalj iz keša, retry download/upload

— Kraj specifikacije —
