## **Sortiranje kartica prijava kvarova** (*Issue card sorting*)

### 1) Osnovno pravilo — deterministički redoslijed
Lista/kanban moraju imati **stabilan algoritam** (*stable deterministic sort*) da kartice ne “skaču” pri refreshu, offline/online prelazu i promjeni filtera.

- **Primarni ključ** (*primary key*): **operativni prioritet** (hitnost + SLA rizik + status)
- **Sekundarni ključ** (*secondary key*): **relevantni datum** (zavisi od statusa)
- **Tercijarni ključ** (*tie-breaker*): **ID** (`id DESC`) ili `createdAt DESC`

---

### 2) Stanar — default redoslijed (*Resident default sort*)

#### 2.1 Slojevi (od vrha ka dnu)
1. **Hitno** (*Urgent*) + **aktivno** (nije zatvoreno)  
2. **Aktivno** (*Active*) — statusi: **Primljeno**, **U obradi**, **U radu**, **Čeka potvrdu**
3. **Reotvoreno** (*Reopened*) — tretira se kao aktivno ali ide iznad ostalih aktivnih (osim hitnih)
4. **Zatvoreno** (*Closed*)
5. **Odbijeno** (*Rejected*) — prikaz samo ako filter uključuje

> Napomena: Stanaru je cilj “šta je u toku” i “šta čeka mene”.

#### 2.2 Prioritet unutar aktivnog sloja
- **Čeka potvrdu** (*Waiting for confirmation*) ide iznad ostalih aktivnih (jer traži akciju stanara)
- zatim **U radu** → **U obradi** → **Primljeno**

#### 2.3 Relevantni datum (*effective date*)
- **Primljeno**: `createdAt DESC`
- **U obradi**: `triagedAt DESC` (fallback `updatedAt`)
- **U radu**: `workStartedAt DESC` (fallback `updatedAt`)
- **Čeka potvrdu**: `readyForApprovalAt DESC`
- **Zatvoreno**: `closedAt DESC`
- **Odbijeno**: `rejectedAt DESC`

#### 2.4 Tie-breaker
Ako su isti sloj i isti datum:
- `id DESC`

---

### 3) Upravnik — default redoslijed (*Manager default sort*)

Upravniku je prioritet: **SLA rizik**, **hitnost**, i **statusna kolona** (kanban).

#### 3.1 Kanban (po kolonama)
Kolone po statusu: **Primljeno** | **U obradi** | **U radu** | **Čeka potvrdu** | **Zatvoreno** | **Odbijeno**

**Unutar svake kolone** redoslijed:
1. **SLA breach** (*breach*)  
2. **SLA warning** (*warning*)  
3. **Hitno** (*urgent*)  
4. **Nove poruke** (*hasNewMessages*)  
5. **Relevantni datum** (vidi 3.2)  
6. Tie-breaker `id DESC`

> Pravilo: **SLA rizik uvijek pobjeđuje hitnost** (jer je breach operativna eskalacija).

#### 3.2 Relevantni datum po statusu
- **Primljeno**: `createdAt DESC` (najnovije prijave gore)
- **U obradi**: `triageDueAt ASC` (najbliži SLA rok gore)  
- **U radu**: `finishDueAt ASC` (najbliži završni rok gore)  
- **Čeka potvrdu**: `readyForApprovalAt ASC` (najduže čekanje gore)  
- **Zatvoreno**: `closedAt DESC`
- **Odbijeno**: `rejectedAt DESC`

#### 3.3 Operativni “boost” (premium)
Premium može dodati dodatne boost faktore (unutar kolone):
- `isCommonArea=true` (zajednički prostor) iznad privatnih (opciono)
- `hasCostApprovalPending=true` (čeka odobrenje troška) iznad ostalih
- `contractorDelayed=true` (partner probio termin) iznad ostalih

Ali i dalje važi: **breach > warning > urgent > newMessages > date > id**.

---

### 4) Partner/Izvođač — default redoslijed (*Vendor default sort*)

Partneru je cilj: “šta radim danas” i “šta gori”.

#### 4.1 Slojevi (od vrha ka dnu)
1. **SLA breach/warning** (ako se prikazuje)  
2. **Današnji termin** (*scheduledFor today*)  
3. **U toku** (*in progress*)  
4. **Prihvaćeno, čeka start**  
5. **Završeno, čeka predaju izvještaja**  
6. **Zatvoreno/Arhiva**

#### 4.2 Relevantni datum
- Ako postoji `scheduledFor`: `scheduledFor ASC` (najbliže prvo)
- Inače: `updatedAt DESC`
- Tie-breaker `id DESC`

---

### 5) Filteri i sortiranje (*Filtering + sorting interaction*)
Filteri **ne mijenjaju algoritam**, samo sužavaju skup.

Tipični filteri:
- Status (kolone ili lista)
- Prioritet (hitno/normalno)
- Kategorija (voda/struja/lift/…)
- Lokacija (zgrada/ulaz/sprat/stan, zajednički prostor)
- Izvođač (partner)
- SLA (ok/warning/breach)
- Period (createdAt ili closedAt zavisno od statusa)

---

### 6) Offline i stabilnost (*Offline + refresh stability*)
- Of
