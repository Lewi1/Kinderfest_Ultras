# Supabase-Anbindung — Ideen & Anwärter

Die Website ist statisch (GitHub Pages) und kann selbst nichts in eine Datenbank
schreiben. Deshalb schreibt das JavaScript im Browser direkt in **Supabase**.

Solange in `index.html` noch die Platzhalter stehen, läuft alles im **Lokal-Modus**
(Ideen werden nur auf dem Gerät gemerkt, das Anwärter-Formular meldet „noch nicht
scharfgeschaltet"). Sobald du unten die zwei Werte einträgst, geht es scharf.

---

## 1. Supabase-Projekt anlegen

1. Auf <https://supabase.com> mit GitHub/Google anmelden (kostenlos).
2. **New project** → Name z. B. `kinderfest-ultras`, Region `Central EU (Frankfurt)`,
   ein Datenbank-Passwort vergeben (brauchst du für die Website **nicht**, nur zum
   Aufheben).
3. Warten, bis das Projekt fertig provisioniert ist (~1–2 Min).

## 2. Tabellen + Sicherheitsregeln anlegen

Links im Menü **SQL Editor** → **New query** → das Folgende komplett einfügen und
**Run** drücken:

```sql
-- ============ Tabelle: Ideen ============
create table if not exists public.ideen (
  id         bigint generated always as identity primary key,
  text       text not null check (char_length(text) between 2 and 200),
  status     text not null default 'neu',   -- neu | approved | abgelehnt
  created_at timestamptz not null default now()
);

alter table public.ideen enable row level security;
grant insert, select on public.ideen to anon;

-- Jeder darf eine Idee EINreichen (nur mit Status 'neu')
create policy "ideen_insert_anon" on public.ideen
  for insert to anon
  with check (char_length(text) between 2 and 200 and status = 'neu');

-- Öffentlich lesbar sind NUR freigegebene Ideen
create policy "ideen_select_approved" on public.ideen
  for select to anon
  using (status = 'approved');


-- ============ Tabelle: Anwärter (privat!) ============
create table if not exists public.anwaerter (
  id          bigint generated always as identity primary key,
  name        text not null check (char_length(name) between 2 and 80),
  kontakt     text not null check (char_length(kontakt) between 3 and 120),
  ort         text check (char_length(ort) <= 120),
  motivation  text not null check (char_length(motivation) between 2 and 1000),
  wetttrinken text,
  status      text not null default 'neu',
  created_at  timestamptz not null default now()
);

alter table public.anwaerter enable row level security;
grant insert on public.anwaerter to anon;

-- Jeder darf sich BEWERBEN (nur einfügen) …
create policy "anwaerter_insert_anon" on public.anwaerter
  for insert to anon
  with check (
    char_length(name) between 2 and 80
    and char_length(motivation) between 2 and 1000
    and status = 'neu'
  );
-- … aber es gibt bewusst KEINE select-Policy für anon:
--    Bewerbungen kann niemand über die Website auslesen,
--    nur du im Supabase-Dashboard (Table Editor).
```

Damit gilt von der Website aus: **nur einfügen**. Niemand kann fremde Bewerbungen
lesen, ändern oder löschen — auch nicht mit dem öffentlichen Schlüssel.

## 3. Die zwei Werte in `index.html` eintragen

Links **Project Settings** (Zahnrad) → **API Keys**. Supabase nutzt inzwischen ein
neues Schlüssel-System (Reiter „Publishable and secret API keys"):

- **Project URL** (unter Settings → **Data API**) → `https://mqqjtjbfywwwiansqmbh.supabase.co`
- **Publishable key** → beginnt mit `sb_publishable_…` → **das ist der Browser-Key**
  (früher hieß der „anon public"). Steht ausdrücklich: „can be safely shared publicly".

In `index.html` ganz oben im `<script>`-Block ist die URL schon eingetragen; nur noch
den Publishable-Key einsetzen:

```js
var SUPABASE_URL      = 'https://mqqjtjbfywwwiansqmbh.supabase.co';  // schon gesetzt
var SUPABASE_ANON_KEY = 'sb_publishable_…';                          // <- Publishable key
```

Speichern, committen, pushen — fertig. Der Publishable-Key ist absichtlich öffentlich;
er darf im Quelltext stehen (die RLS-Regeln oben sind der Schutz). Nimm **niemals**
den **Secret key** (`sb_secret_…`) — der umgeht alle Regeln und gehört nur auf einen
Server.

## 4. So schaust du rein (Vorstand)

Am bequemsten über den **Mitgliederbereich** (`intern.html`, siehe Schritt 5).
Alternativ direkt in Supabase:

- **Ideen sichten:** Table Editor → `ideen`. Was ihr übernehmen wollt: Feld
  `status` auf `approved` setzen → erscheint automatisch auf der Website unter
  „Ideenspeicher 2027". Auf `abgelehnt` = bleibt privat.
- **Bewerbungen:** Table Editor → `anwaerter`. Neue Einträge stehen auf `neu`.

---

## 5. Mitgliederbereich mit Login (`intern.html`)

Damit der Vorstand Bewerbungen sehen und Ideen freigeben/löschen kann — **nur
eingeloggt**. Wir nutzen Supabase **Auth** (E-Mail + Passwort). Zugriff bekommt
nur, wer auf der **Vorstands-Allowlist** steht — so sind die Bewerber-Daten (Name,
Kontakt) auch dann geschützt, wenn versehentlich jemand ein Konto anlegt.

### 5a. Öffentliche Registrierung ausschalten (wichtig!)

**Authentication → Sign In / Providers → Email**: „Allow new users to sign up"
**deaktivieren**. Sonst könnte sich jeder ein Konto anlegen. Accounts legst du
selbst an (5c).

### 5b. Allowlist + Regeln anlegen (SQL Editor → Run)

```sql
-- Wer darf in den Mitgliederbereich? (E-Mail-Liste)
create table if not exists public.vorstand (email text primary key);
alter table public.vorstand enable row level security;   -- niemand darf sie über die API lesen

-- Prüf-Funktion: ist die eingeloggte E-Mail auf der Liste?
create or replace function public.is_vorstand()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (
    select 1 from public.vorstand v
    where lower(v.email) = lower(auth.jwt() ->> 'email')
  );
$$;
revoke all on function public.is_vorstand() from public, anon;
grant execute on function public.is_vorstand() to authenticated;

-- Ideen: Vorstand darf ALLES sehen, freigeben (update), löschen
grant select, update, delete on public.ideen to authenticated;
create policy "ideen_select_vorstand" on public.ideen for select to authenticated using (public.is_vorstand());
create policy "ideen_update_vorstand" on public.ideen for update to authenticated using (public.is_vorstand()) with check (public.is_vorstand());
create policy "ideen_delete_vorstand" on public.ideen for delete to authenticated using (public.is_vorstand());

-- Anwärter: Vorstand darf sehen, Status setzen (update), löschen
grant select, update, delete on public.anwaerter to authenticated;
create policy "anwaerter_select_vorstand" on public.anwaerter for select to authenticated using (public.is_vorstand());
create policy "anwaerter_update_vorstand" on public.anwaerter for update to authenticated using (public.is_vorstand()) with check (public.is_vorstand());
create policy "anwaerter_delete_vorstand" on public.anwaerter for delete to authenticated using (public.is_vorstand());
```

### 5c. Vorstands-Accounts anlegen

Für jede Person zwei Schritte:

1. **E-Mail auf die Allowlist** (SQL Editor):
   ```sql
   insert into public.vorstand (email) values ('leo.wildmoser@gmail.com');
   -- weitere: ('philip@example.com'), ('...') ...
   ```
2. **Konto anlegen:** Authentication → **Users** → **Add user** → E-Mail + Passwort
   eintragen und **„Auto Confirm User" anhaken** (sonst muss die E-Mail erst
   bestätigt werden). Die E-Mail muss exakt der aus Schritt 1 entsprechen.

### 5d. Nutzen

`https://kinderfestultras.de/intern.html` (oder Footer-Link **„Mitglieder"**) →
anmelden → Bewerbungen ansehen/annehmen/ablehnen/löschen, Ideen freigeben (=
öffentlich)/ablehnen/löschen. Wer eingeloggt, aber **nicht** auf der Allowlist ist,
bekommt „Kein Zugriff".

---

## 6. Wetttrinken-Termine in der Datenbank

Damit gebuchte Duell-Termine **für alle** sichtbar sind (nicht nur pro Gerät) und
kein Termin doppelt vergeben wird. SQL Editor → Run:

```sql
-- ============ Tabelle: Wetttrinken-Termine ============
create table if not exists public.wetttrinken (
  id         bigint generated always as identity primary key,
  name       text not null check (char_length(name) between 2 and 40),
  slot       text not null unique check (char_length(slot) between 3 and 60),
  created_at timestamptz not null default now()
);

alter table public.wetttrinken enable row level security;
grant insert, select on public.wetttrinken to anon;
grant select, delete on public.wetttrinken to authenticated;

-- Jeder darf einen freien Termin buchen (slot ist UNIQUE -> doppelt geht nicht)
create policy "wett_insert_anon" on public.wetttrinken
  for insert to anon
  with check (char_length(name) between 2 and 40 and char_length(slot) between 3 and 60);

-- Scoreboard/Belegung ist oeffentlich lesbar
create policy "wett_select_anon" on public.wetttrinken
  for select to anon using (true);

-- Vorstand darf Termine sehen & loeschen (Duell absagen -> Slot wird wieder frei)
create policy "wett_select_vorstand" on public.wetttrinken
  for select to authenticated using (public.is_vorstand());
create policy "wett_delete_vorstand" on public.wetttrinken
  for delete to authenticated using (public.is_vorstand());
```

Danach ist die Terminbuchung auf der Startseite zentral: gebuchte Slots sind sofort
bei allen als „belegt" durchgestrichen, und im Mitgliederbereich (`intern.html`)
kann der Vorstand Termine absagen.

---

## 7. Nur-Lese-Login für Mitglieder

Ein Login, das **alles sehen**, aber **nichts ändern/löschen/freigeben** darf
(z. B. `mitglieder@kinderfestultras.de` für die Stammmitglieder). Zweite Allowlist
`mitglieder`; Lesen für Vorstand **und** Mitglieder, Ändern bleibt beim Vorstand.
SQL Editor → Run:

```sql
-- ============ Read-only Mitglieder ============
create table if not exists public.mitglieder (email text primary key);
alter table public.mitglieder enable row level security;

-- Wer darf LESEN? (Vorstand ODER read-only Mitglied)
create or replace function public.is_mitglied()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.vorstand   v where lower(v.email) = lower(auth.jwt() ->> 'email'))
      or exists (select 1 from public.mitglieder m where lower(m.email) = lower(auth.jwt() ->> 'email'));
$$;
revoke all on function public.is_mitglied() from public, anon;
grant execute on function public.is_mitglied() to authenticated;

-- SELECT jetzt fuer alle Mitglieder; UPDATE/DELETE bleiben (aus Schritt 5b/6) beim Vorstand
drop policy if exists "ideen_select_vorstand"     on public.ideen;
create policy "ideen_select_mitglied"     on public.ideen      for select to authenticated using (public.is_mitglied());

drop policy if exists "anwaerter_select_vorstand" on public.anwaerter;
create policy "anwaerter_select_mitglied" on public.anwaerter  for select to authenticated using (public.is_mitglied());

drop policy if exists "wett_select_vorstand"      on public.wetttrinken;
create policy "wett_select_mitglied"      on public.wetttrinken for select to authenticated using (public.is_mitglied());

-- Read-only Login auf die Liste
insert into public.mitglieder (email) values ('mitglieder@kinderfestultras.de')
on conflict (email) do nothing;
```

Dann noch (wie beim Vorstand) einen **Auth-Account** anlegen: Authentication →
Users → Add user → `mitglieder@kinderfestultras.de` + Passwort, „Auto Confirm" an.
Wer sich damit einloggt, sieht alles, aber **ohne** Aktions-Buttons — und selbst
wenn jemand tricksen wollte, blockt die Datenbank jedes Ändern (RLS).

> Wichtig: Diese E-Mail gehört in `mitglieder`, **nicht** in `vorstand`. Steht eine
> Adresse in beiden, gewinnt Vorstand (darf dann doch alles).

---

## 8. Besucherzähler (unique + Gesamt-Aufrufe)

Zeigt im Footer, wie viele Leute die Seite besucht haben. **Keine personenbezogenen
Daten, keine IP** — nur eine zufällige ID pro Gerät (im Browser gespeichert). SQL
Editor → Run:

```sql
-- ============ Besuche zaehlen ============
create table if not exists public.besuche (
  id         bigint generated always as identity primary key,
  visitor    uuid not null,
  created_at timestamptz not null default now()
);

alter table public.besuche enable row level security;
grant insert on public.besuche to anon;

-- Jeder darf einen Besuch melden (nur einfuegen, NICHT lesen -> kein Log auslesbar)
create policy "besuche_insert_anon" on public.besuche
  for insert to anon with check (true);

-- Aggregierte Zahlen (gesamt + unique), oeffentlich abrufbar ohne Rohdaten
create or replace function public.besucher_stats()
returns json language sql stable security definer set search_path = public as $$
  select json_build_object(
    'gesamt', count(*),
    'unique', count(distinct visitor)
  ) from public.besuche;
$$;
revoke all on function public.besucher_stats() from public;
grant execute on function public.besucher_stats() to anon;
```

- **gesamt** = jeder Seitenaufruf (auch Reloads).
- **unique** = verschiedene Geräte (über die Zufalls-ID).
- Die Rohdaten (`besuche`) kann niemand über die Website auslesen — nur die
  fertige Summe über die Funktion.

---

## Optional: Spam-Bremse

## Optional: Spam-Bremse

Es gibt schon ein verstecktes Honeypot-Feld (füllt nur ein Bot aus → wird
verworfen). Wenn doch mal Müll reinkommt, kannst du in Supabase die Zeilen
einfach löschen oder unter **Authentication → Rate Limits** / **Edge Functions**
nachschärfen. Für ein Vereinsfest reicht der Honeypot i. d. R.
