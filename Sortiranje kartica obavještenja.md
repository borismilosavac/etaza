## **Sortiranje kartica obavještenja** (*Notification card sorting*)

### 1) Osnovno pravilo — deterministički redoslijed
Feed mora imati **jedan stabilan algoritam** (*stable deterministic sort*) da se kartice ne “premeću” pri refreshu, offline/online prelazu i promjeni filtera.

- **Primarni ključ** (*primary key*): **prioritet prikaza**
- **Sekundarni ključ** (*secondary key*): **relevantni datum**
- **Tercijarni ključ** (*tie-breaker*): **ID** (ili `createdAt`) radi stabilnosti

---

### 2) Stanar — default redoslijed (*Resident default sort*)

#### 2.1 Slojevi (od vrha ka dnu)
1. **Pinovana hitna** (*Pinned urgent*) — samo dok traje pin (24/48h)
2. **Hitna** (*Urgent*) — nepinovana ili nakon isteka pina
3. **Važna** (*Important*, premium) — unutar istog sloja iznad ostalih
4. **Nepročitana** (*Unread*)
5. **Pročitana** (*Read*)
6. **Arhivirana** (*Archived*) — prikazuje se samo ako je aktivan filter “Arhiva”

> Napomena: **Važna** je “boost” unutar sloja, ali ne prelazi preko **Pinovane hitne** i **Hitne**.

#### 2.2 Relevantni datum (*effective date*)
- Ako je status **Zakazano** (*Scheduled*): koristi `scheduleAt`
- Ako je status **Objavljeno/Aktivno** (*Active*): koristi `publishedAt`
- Ako je status **Arhiva** (*Archived*): koristi `archivedAt` (fallback `publishedAt`)

**Smjer sortiranja**: opadajuće (*descending*): novije ide gore.

#### 2.3 Tie-breaker
Ako su isti sloj i isti datum:
- `id DESC` (ili `createdAt DESC` ako je stabilnije u bazi)

---

### 3) Upravnik — default redoslijed (*Manager default sort*)

#### 3.1 Prioritet po statusima (upravljanje sadržajem)
1. **Draft** (*Nacrt*) — `updatedAt DESC`
2. **Zakazano** (*Scheduled*) — `scheduleAt ASC` (najbliže objave gore)
3. **Aktivno** (*Active*) — `publishedAt DESC`
4. **Povučeno** (*Retracted*) — `retractedAt DESC`
5. **Arhiva** (*Archived*) — `archivedAt DESC`

#### 3.2 Operativni “boost” (premium)
Ako postoji indikator **novih poruka stanara** (*new resident messages*):
- unutar istog statusa, kartice sa `hasNewMessages=true` idu iznad ostalih,
- ali **ne prelaze** preko globalnog redoslijeda statusa.

---

### 4) Filteri i sortiranje (*Filtering + sorting interaction*)
Filteri **ne mijenjaju algoritam**, samo sužavaju skup.

- Filter **Nepročitano** (*Unread*): prikazuje samo relevantne kartice, sort po `publishedAt DESC`
- Filter **Hitno** (*Urgent*): prvo **pinovana hitna**, zatim ostale hitne po `publishedAt DESC`
- Filter **Period** (*Time range*): filtrira po `publishedAt` (ili `scheduleAt` za zakazano), zatim primijeni sort pravila

---

### 5) Offline i stabilnost (*Offline + refresh stability*)
- Offline koristi keširane statuse/datume.
- Po povratku online radi se **merge** (*merge strategy*) i ponovna primjena istog sorta.
- Obavezno koristiti **stabilno sortiranje** (*stable sort*) i isti tie-breaker da ne dođe do “skakanja”.

---

### 6) Grupisanje po datumu (*Date grouping*) — opcionalno
Ako UI koristi sekcije (“Danas”, “Ove sedmice”, “Ranije”):
- sekcija se određuje po istom **relevantnom datumu** (`scheduleAt`/`publishedAt`/`archivedAt`)
- unutar sekcije važe ista pravila slojeva (pin/hitno/važno/nepročitano…)

---

### 7) Pristupačnost (*Accessibility*)
- Promjena reda (npr. istek pina) mora imati **nenametljivu najavu** (*aria-live=polite*) i vizuelni indikator (npr. badge “Pin istekao” 1x).
- Fokus nakon refresh-a ostaje na istoj kartici (po **ID**), ne skače na prvu u listi.
