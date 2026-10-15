# Supabase duomenų bazė

Šiame dokumente aprašyti Supabase objektai, kuriuos projektas naudoja arba kurių reikia jo veikimui. Programėlė pati lentelių nesukuria: jas ir RLS politikas reikia paruošti Supabase projekte.

## Naudojami objektai

| Objektas | Schema / tipas | Naudojimas |
|---|---|---|
| `auth.users` | Supabase Auth sistemos lentelė | Registracija, prisijungimas, sesijos ir naudotojo UUID. Programėlė paskyras tvarko per `supabase.auth`, o ne tiesioginėmis SQL užklausomis. |
| `public.users` | Lentelė (profilio įrašai) | README nurodo, kad įrašą sukuria DB trigeris, kai sukuriamas `auth.users` naudotojas. Frontend jos tiesiogiai nenaudoja. |
| `public.habits` | Lentelė | Naudotojų įpročių kūrimas, skaitymas, redagavimas ir trynimas. |
| `public.habit_logs` | Lentelė | Įpročio atlikimo konkrečią kalendorinę dieną žymėjimas ir istorija. |

## Lentelių laukai

Tipai ir numatytosios reikšmės žemiau yra rekomenduojama schema pagal programos užklausas. Projekte nėra migracijų, kurios patvirtintų tikslius jau sukurtų lentelių tipus.

### `public.users`

Profilių lentelė, susieta su Supabase Auth naudotojais. Ji minima projekto README, bet nėra tiesiogiai skaitoma ar keičiama programos kode.

| Laukas | PostgreSQL tipas | Apribojimai / numatytoji reikšmė | Aprašymas |
|---|---|---|---|
| `id` | `uuid` | Pirminis raktas; FK į `auth.users(id)` su `ON DELETE CASCADE` | Profilio ID, sutampantis su Auth naudotojo ID. |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` | Profilio sukūrimo laikas. |

Trigeris turi sukurti `public.users` įrašą naujam `auth.users` naudotojui. Jei profilio lentelės nereikia kitoms jūsų sistemos dalims, pati ši programėlė nuo jos nepriklauso.

### `public.habits`

| Laukas | PostgreSQL tipas | Apribojimai / numatytoji reikšmė | Aprašymas |
|---|---|---|---|
| `id` | `uuid` | Pirminis raktas; `NOT NULL DEFAULT gen_random_uuid()` | Unikalus įpročio ID. |
| `user_id` | `uuid` | `NOT NULL`; FK į `auth.users(id)` su `ON DELETE CASCADE` | Įpročio savininkas. Kuriant įprotį programėlė čia įrašo prisijungusio naudotojo ID. |
| `name` | `text` | `NOT NULL`; rekomenduojama `CHECK (char_length(name) BETWEEN 1 AND 100)` | Įpročio pavadinimas. UI pašalina priekinį ir galinį tarpą, reikalauja ne tuščios reikšmės, leidžia iki 100 simbolių. |
| `description` | `text` | `NULL` leidžiamas; rekomenduojama `CHECK (description IS NULL OR char_length(description) <= 500)` | Pasirenkamas aprašymas. Tuščią reikšmę programėlė įrašo kaip `NULL`; daugiausia 500 simbolių. |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` | Sukūrimo laikas; sąrašas rikiuojamas nuo seniausio. |

### `public.habit_logs`

| Laukas | PostgreSQL tipas | Apribojimai / numatytoji reikšmė | Aprašymas |
|---|---|---|---|
| `id` | `uuid` | Pirminis raktas; `NOT NULL DEFAULT gen_random_uuid()` | Unikalus atlikimo įrašo ID. |
| `habit_id` | `uuid` | `NOT NULL`; FK į `public.habits(id)` su `ON DELETE CASCADE` | Įprotis, kurio atlikimas užregistruotas. |
| `date` | `date` | `NOT NULL` | Atlikimo diena pagal naudotojo vietinį kalendorių (`YYYY-MM-DD`). |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` | Atlikimo įrašo sukūrimo laikas. |

Privalomas unikalumo apribojimas: `UNIQUE (habit_id, date)`. Programėlė dublikatą (PostgreSQL klaidos kodas `23505`) laiko jau atliktu įpročiu ir grąžina esamą įrašą.

## Ryšiai ir duomenų gyvavimo ciklas

```text
auth.users (1) ─── (0 arba 1) public.users
auth.users (1) ─── (N)       public.habits
public.habits (1) ─── (N)    public.habit_logs
```

- `public.users.id` sutampa su `auth.users.id`; profilio įrašą sukuria trigeris.
- `public.habits.user_id` nurodo į `auth.users.id` ir apibrėžia įpročio savininką.
- `public.habit_logs.habit_id` nurodo į įprotį. Įrašo ištrynimas kartu su įpročiu vyksta per `ON DELETE CASCADE`.
- Vienam įpročiui gali būti daug atlikimo įrašų, tačiau konkrečiai dienai leidžiamas tik vienas.
- Įpročio atlikimas įrašomas įterpiant `habit_logs` eilutę, o atžymėjimas – ją ištrinant.
- Įpročių sąrašas rodomas pagal `created_at` didėjimo tvarka. Dienos istorija nuskaitoma pagal `habit_id` ir datos intervalą.

## RLS (Row Level Security)

Įjunkite RLS bent `public.habits` ir `public.habit_logs` lentelėms, nes naršyklėje naudojamas anon / publishable raktas. Programos užklausos įpročius filtruoja pagal `user_id`; tai naudinga, bet pati savaime nėra prieigos kontrolė – ją turi užtikrinti RLS.

Reikalingų politikų elgsena `authenticated` vaidmeniui:

- `habits` SELECT, INSERT, UPDATE, DELETE: leidžiama tik kai įrašo `user_id = auth.uid()`. INSERT politikoje tikrinkite `WITH CHECK`; UPDATE atveju užtikrinkite tiek esamos eilutės matomumą (`USING`), tiek naujos reikšmės savininką (`WITH CHECK`).
- `habit_logs` SELECT, INSERT, DELETE: leidžiama tik jei susietas `habits` įrašas priklauso `auth.uid()` naudotojui. UPDATE šiuo metu programai nereikalingas.
- `public.users`: jei lentelę paliekate, profilio kūrimą vykdykite saugiu DB trigeriu. Klientui nesuteikite galimybės kurti profilio kitam naudotojui; papildomas profilio skaitymo / redagavimo teises apibrėžkite pagal poreikį.

## Konfigūracija ir ko nereikia kurti rankiniu būdu

- Klientas konfigūruojamas `VITE_SUPABASE_URL` ir `VITE_SUPABASE_ANON_KEY` reikšmėmis. `service_role` / secret rakto negalima dėti į frontend aplinkos kintamuosius.
- Slaptažodžius ir autentifikavimo paskyras valdo Supabase Auth; jų nelaikykite `public.users` lentelėje.
- Registracija naudoja el. paštą ir slaptažodį. Programėlės UI reikalauja bent 8 simbolių slaptažodžio; el. pašto patvirtinimo elgsena priklauso nuo Supabase Auth nustatymo.
- Programėlė `public.users` lentelės neįterpia tiesiogiai. README reikalauja, kad paskyros profilio eilutę sukurtų DB trigeris.
- Datos `habit_logs.date` laukui skaičiuojamos naudotojo vietos laiku ir saugomos kaip SQL `date`, o ne UTC laiko žyma.
