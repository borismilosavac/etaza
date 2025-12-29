# **Frontend specifikacija (Flutter)** — Modul **Obavještenja** (*Notifications*) 

> [!IMPORTANT]
> **Cilj** (*Goal*): Ovo je implementacioni dokument za izradu **frontenda** u Flutteru (Android/iOS + Web), za role **Stanar** (*Resident*) i **Upravnik** (*Manager*). Uključuje UI, UX, stanja, validacije, rute, komponente, state management, offline, analytics, accessibility, i jasno razdvajanje **Freemium** i **Premium** capability‑a.

## Sadržaj

* [0. Scope i definicije](#0-scope-i-definicije)
* [1. Feature flagovi i plan switch](#1-feature-flagovi-i-plan-switch)
* [2. Informaciona arhitektura i rute](#2-informaciona-arhitektura-i-rute)
* [3. Data modeli (frontend)](#3-data-modeli-frontend)
* [4. State management (Riverpod/Bloc)](#4-state-management-riverpodbloс)
* [5. UI komponente i widget spec](#5-ui-komponente-i-widget-spec)
* [6. Ekrani — Stanar](#6-ekrani--stanar)
* [7. Ekrani — Upravnik](#7-ekrani--upravnik)
* [8. Filteri, sortiranje, paginacija](#8-filteri-sortiranje-paginacija)
* [9. Poruke (thread) i prikaz više stanara](#9-poruke-thread-i-prikaz-više-stanara)
* [10. Kreiranje obavještenja (wizard 4 koraka)](#10-kreiranje-obavještenja-wizard-4-koraka)
* [11. Bulk akcije (upravnik)](#11-bulk-akcije-upravnik)
* [12. Notifikacije (push/email) i deep link](#12-notifikacije-pushemail-i-deep-link)
* [13. Upload priloga](#13-upload-priloga)
* [14. Offline, caching i retry](#14-offline-caching-i-retry)
* [15. Validacije i error handling](#15-validacije-i-error-handling)
* [16. Accessibility (WCAG 2.1 AA)](#16-accessibility-wcag-21-aa)
* [17. Analitika (eventi)](#17-analitika-eventi)
* [18. QA checklist (frontend)](#18-qa-checklist-frontend)

---

## 0. Scope i definicije

### 0.1 Uloge

* **Stanar** (*Resident*) — pregled, filtriranje, čitanje, potvrda prijema (ako aktivno), poruka upravniku.
* **Upravnik** (*Manager*) — lista + filteri + bulk, kreiranje/zakazivanje, ciljanje, prilozi, pregled poruka od više stanara, odgovori.

### 0.2 Planovi

* **Freemium**: sve kritične funkcije za normalan rad zgrade (feed, detalji, poruke, osnovno zakazivanje, osnovni kanali).
* **Premium**: napredno zakazivanje (rekurencija), SMS/failover, automatizacije, napredna analitika, napredni export.

### 0.3 Kanali komunikacije

* **In-app** (obavezno)
* **Push** (ako postoji device token i dozvola)
* **Email** (opt-in; opcionalan podatak)
* **SMS** (Premium; opt-in; opcionalan podatak)

---

## 1. Feature flagovi i plan switch

### 1.1 Feature flags (kanonsko)

Frontend mora raditi isključivo na osnovu capability seta koji dolazi iz backend-a.

```json
{
  "plan": "freemium|premium",
  "features": {
    "notifications": {
      "recurrence": false,
      "sms": false,
      "failover": false,
      "automation": false,
      "advancedAnalytics": false,
      "advancedExport": false,
      "markImportant": false
    }
  }
}
```

### 1.2 UI “switch” (Freemium/Premium)

**Namjena**: Figma prototip i demo režim. U produkciji se ne koristi kao ručni toggle (osim ako postoji “preview mode” za prodaju).

* Ako `plan=freemium`: premium UI se prikazuje kao **disabled + lock icon + tooltip**.
* Ako `plan=premium`: polja su aktivna.

**Komponenta**: `PlanSwitch(mode: server|demo)`

---

## 2. Informaciona arhitektura i rute

### 2.1 Global rute

* `/notices` — lista obavještenja
* `/notices/:id` — detalji

### 2.2 Upravnik rute

* `/notices/new` — wizard kreiranja
* `/notices/:id/edit` — edit draft (ako dozvoljeno)
* `/notices/:id/preview` — preview (web: side panel)

### 2.3 Deep link rute

* `zgradaplus://notices/:id`
* `zgradaplus://notices/:id?focus=thread&messageId=...`

---

## 3. Data modeli (frontend)

> [!NOTE]
> Modeli su frontend view-modeli. Backend DTO može biti širi. Držati striktne tipove i null-safety.

### 3.1 Enumi

```dart
enum NoticeStatus { draft, scheduled, published, archived, retracted }
enum NoticeChannel { inApp, push, email, sms }
enum NoticePriority { normal, urgent }
enum VisibilityScope { building, segment, unit, privateThread }
```

### 3.2 Core modeli

```dart
class Notice {
  final String id;
  final String title;
  final String bodyMd;
  final NoticeStatus status;
  final NoticePriority priority;
  final DateTime createdAt;
  final DateTime updatedAt;
  final DateTime? publishedAt;
  final DateTime? scheduledAt;
  final DateTime? archivedAt;
  final Targeting targeting;
  final List<NoticeChannel> channels;
  final List<Attachment> attachments;
  final NoticeMetrics metrics;
  final bool isPinned; // manager-controlled
  final bool isImportant; // tenant premium optional
  final bool hasNewMessages; // for manager list badge
}

class Targeting {
  final String buildingId;
  final List<String>? entrances;
  final List<int>? floors;
  final List<String>? groups;
  final List<String>? unitIds;
}

class Attachment {
  final String id;
  final String name;
  final String url;
  final String mime;
  final int sizeBytes;
  final AttachmentKind kind; // image/pdf/video/link
}

enum AttachmentKind { image, pdf, video, link }

class NoticeMetrics {
  final int views;
  final int acks;
  final int replies;
}
```

### 3.3 Thread modeli

```dart
class ThreadMessage {
  final String id;
  final String noticeId;
  final String authorId;
  final AuthorRole authorRole; // tenant/manager
  final String body;
  final List<Attachment> attachments;
  final DateTime createdAt;
  final bool isDeleted; // soft
}

enum AuthorRole { tenant, manager }

class TenantThreadGroup {
  final String tenantKey; // unitLabel ili uid
  final String displayName; // "B-204" ili "B-204 (Mira)"
  final bool hasContact;
  final List<ThreadMessage> messages;
  final int unreadCount;
  final DateTime lastMessageAt;
}
```

### 3.4 Read/Ack

```dart
class ReadReceipt {
  final String noticeId;
  final DateTime? readAt;
  final bool acknowledged;
  final DateTime? acknowledgedAt;
}
```

---

## 4. State management (Riverpod/Bloc)

### 4.1 Obavezno pravilo

Koristiti **jedan** state management pattern u cijeloj aplikaciji.

### 4.2 Predložena struktura (Riverpod)

* `noticesRepositoryProvider`
* `noticesListControllerProvider(NoticeListQuery)`
* `noticeDetailControllerProvider(noticeId)`
* `threadControllerProvider(noticeId)`
* `composerControllerProvider(noticeId, role)`

### 4.3 Stanja (sealed)

```dart
sealed class AsyncState<T> {
  const AsyncState();
}
class Loading<T> extends AsyncState<T> {}
class Data<T> extends AsyncState<T> { final T value; }
class Error<T> extends AsyncState<T> { final AppError error; }

class AppError {
  final String code; // e.g. NET_OFFLINE, VAL_TITLE, AUTH_PLAN_REQUIRED
  final String message;
  final bool retryable;
}
```

---

## 5. UI komponente i widget spec

### 5.1 Tokens i Theme

Svi widgeti moraju koristiti dizajn tokene (iz `design-tokens.tokens.json`) mapirane u Flutter `ThemeData`.

* Primary CTA: `blue.600` / hover `blue.700` / focus ring `blue.300`
* Card: bg `gray modern.50`, border `200`, radius `16`
* Status badges: `success/warning/error` tints

### 5.2 Osnovni widgeti

* `NoticeListAppBar`
* `NoticeFilterBar` (web sticky)
* `NoticeCard`
* `NoticeDetailHeader`
* `NoticeBodyMarkdown`
* `AttachmentStrip` + `AttachmentTile`
* `AckButton`
* `ReplyCtaButton` (tenant)
* `ThreadList` (tenant)
* `ThreadGroupsPanel` (manager)
* `MessageComposerSheet` (mobile) / `MessageComposerPanel` (web)
* `PlanSwitch` (demo)
* `PremiumGate` (lock)

### 5.3 Widget API (obavezno)

```dart
class NoticeCard extends StatelessWidget {
  final Notice notice;
  final bool isSelected;
  final VoidCallback onOpen;
  final VoidCallback? onToggleSelect; // manager bulk
}

class PremiumGate extends StatelessWidget {
  final bool enabled;
  final Widget child;
  final String lockedTooltip;
}
```

---

## 6. Ekrani — Stanar

### 6.1 Lista obavještenja — `/notices`

**Layout**:

* Mobile: `SliverAppBar` + list
* Web: 2‑kolone (lista lijevo, detalj desno) ako širina ≥ 1024

**Elementi**:

* Search input (debounce 300 ms)
* Filter chips: Status, Period, Hitno, Nepročitano
* Sort label (read-only): po pravilima iz specifikacije

**Stanja**:

* Loading: skeleton 6 kartica
* Empty: edukativni tekst + refresh
* Error: retry

**Interakcije**:

* Tap card → otvara detalj
* Long press (mobile) ne koristi selection (samo manager ima bulk)

### 6.2 Detalj obavještenja — `/notices/:id`

**Sekcije**:

1. Header: naslov, badges (hitno, novo), meta (datum, autor)
2. Body: markdown
3. Attachments
4. CTA zona:

   * `AckButton` (ako je uključeno)
   * `ReplyCtaButton` = **Odgovori upravniku**
5. Thread (stanar↔upravnik)

**CTA ponašanje — “Odgovori upravniku”**:

* Mobile: otvara `MessageComposerSheet` (bottom sheet, draggable)
* Web: otvara `MessageComposerPanel` (desni panel)

**Composer polja**:

* TextField (min 1, max 1000)
* Attachments: max 2 slike + 1 PDF
* Submit: disabled dok nema teksta ili fajl još uploaduje

**After submit**:

* optimistic insert u thread
* toast: “Poruka poslana”
* error: poruka ostaje u outboxu sa retry

### 6.3 Freemium vs Premium (Stanar)

Freemium:

* feed, detalj, reply, ack, osnovni filteri
  Premium:
* opcionalno `markImportant` (ako feature uključen): star/flag u detalju + filter “Važno”

---

## 7. Ekrani — Upravnik

### 7.1 Lista — `/notices` (role=manager)

**Layout**:

* Web: master–detail obavezno
* Mobile: lista + detail push

**Top bar**:

* Create button (primary)
* Bulk selection toggle (checkboxes)
* Plan switch (demo)

**Filters**:

* Status: Draft/Scheduled/Published/Archived
* Scope: building/segment
* Has messages (badge)

**Selection**:

* checkbox na kartici
* sticky bulk bar kada je ≥1 selektovano

### 7.2 Detalj — `/notices/:id`

**Sekcije**:

* Header + akcije (edit draft, archive, export)
* Body + attachments
* Analytics strip (premium gate)
* Thread area:

  * `ThreadGroupsPanel` (lijevo): lista stanara koji su poslali poruke
  * `ThreadDetailPanel` (desno): poruke od izabranog stanara

### 7.3 Prikaz poruka od više stanara

**Group by**: `tenantKey` (unit label / uid)

* sort groups: `unreadCount DESC`, zatim `lastMessageAt DESC`
* badge: unread count
* quick filters: “Neodgovoreno”, “Hitno”, “Sa prilogom”

**Odgovor upravnika**:

* composer u thread detail panelu
* quick replies (template) — premium

---

## 8. Filteri, sortiranje, paginacija

### 8.1 Sort (stanar)

Slojevi: pin urgent → urgent → important(premium) → unread → read → archived.
Datumi: scheduledAt/publishedAt/archivedAt.
Tie: id desc.

### 8.2 Sort (upravnik)

Status layer: draft → scheduled (ASC) → published (DESC) → retracted → archived.
Within status: hasNewMessages boost (ne prelazi layer).

### 8.3 Paginacija

* list: cursor pagination (pageSize 20)
* thread: cursor pagination (pageSize 30), reverse list (najnovije dole)

---

## 9. Poruke (thread) i prikaz više stanara

### 9.1 Pravila privatnosti

Thread je privatni komunikacioni kanal. Ne prikazuje PII (telefon/email) ako nije dostupno i dozvoljeno.

### 9.2 Bulk brisanje poruka

**Nije dozvoljeno**. Samo soft delete pojedinačno (manager), audit obavezan.

### 9.3 Moderacija

* rate limit stanar: 10/min
* zabraniti upload exe/zip

---

## 10. Kreiranje obavještenja (wizard 4 koraka)

> Upravnik: web + mobile.

### 10.1 Korak 1 — Sadržaj

* Title
* Body (markdown)
* Attachments

### 10.2 Korak 2 — Meta / Ciljanje

* Building (default)
* Segment: ulaz/sprat/grupa
* Preview audience size (approx)

### 10.3 Korak 3 — Dodatno

Freemium:

* channels: inApp + push + email (ako postoji)
* schedule one-time

Premium:

* recurrence (weekly/monthly)
* sms channel
* failover rules

### 10.4 Korak 4 — Pregled i objava

* Preview card (mobile/web)
* Confirm

---

## 11. Bulk akcije (upravnik)

Freemium:

* mark as read (manager view)
* archive
* export PDF

Premium:

* export CSV
* bulk schedule changes

---

## 12. Notifikacije (push/email) i deep link

### 12.1 Push payload

* `noticeId`
* `type=new_notice|new_message`
* `messageId` (opciono)

### 12.2 Deep link handling

* Ako `type=new_message`: otvori detail → scroll to message → highlight 3 sek.

---

## 13. Upload priloga

### 13.1 Tipovi

* image: jpg/png
* pdf
* video (manager only, premium optional)

### 13.2 UX

* progress per file
* retry per file
* fail partial allowed

---

## 14. Offline, caching i retry

* Cache list: last 50 notices
* Cache detail: last 20 opened
* Outbox: unsent messages queue

---

## 15. Validacije i error handling

* Title 3–120
* Body ≤ 10k
* scheduleAt ≥ now+2min
* attachments ≤ 10 MB (freemium), 25 MB (premium)

Errors:

* `AUTH_PLAN_REQUIRED` → premium gate
* `NET_OFFLINE` → show offline banner + queue

---

## 16. Accessibility (WCAG 2.1 AA)

* focus ring
* text scaling
* semantics labels
* keyboard shortcuts web: Ctrl+Enter send, Esc close

---

## 17. Analitika (eventi)

* notice_open
* notice_reply_send
* notice_ack
* filter_use
* attachment_download
* manager_publish

---

## 18. QA checklist (frontend)

* push deep link opens correct notice
* message send offline queues then sends
* premium locked fields show lock+tooltip
* multi-tenant messages view works
* sort stable

— Kraj specifikacije —
