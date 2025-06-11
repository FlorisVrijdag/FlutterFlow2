# Hier in het dorp

Een FlutterFlow-project dat Nederlandse dorpjes op een interactieve kaart toont. Deze handleiding beschrijft hoe je Supabase configureert en welke FlutterFlow-schermen je nodig hebt.

## Stap 1: Supabase Database Schema

Maak onderstaande tabellen en relaties aan in Supabase. Gebruik UUID's als primary keys en koppel waar nodig aan `auth.users`.

```sql
-- tabel villages
create table if not exists public.villages (
  id uuid primary key default uuid_generate_v4(),
  name text not null,
  location_lat float8 not null,
  location_lng float8 not null,
  description text,
  image_url text,
  added_by uuid references auth.users(id),
  created_at timestamp with time zone default now()
);

-- tabel sights
create table if not exists public.sights (
  id uuid primary key default uuid_generate_v4(),
  village_id uuid references public.villages(id) on delete cascade,
  title text,
  description text,
  image_url text
);

-- tabel ratings
create table if not exists public.ratings (
  id uuid primary key default uuid_generate_v4(),
  village_id uuid references public.villages(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  rating integer check (rating >= 1 and rating <= 5),
  comment text,
  created_at timestamp with time zone default now()
);

-- optionele favorites tabel
create table if not exists public.favorites (
  user_id uuid references auth.users(id) on delete cascade,
  village_id uuid references public.villages(id) on delete cascade,
  primary key (user_id, village_id)
);
```

Activeer Row Level Security op elke tabel en maak policies die enkel de eigenaar laten toevoegen of aanpassen waar nodig.

## Stap 2: FlutterFlow-schermen en navigatie

1. **HomeScreen**
   - Widget: `GoogleMap`.
   - Query alle `villages` en toon een `Marker` voor elke rij.
   - Bij `onTap` van een marker navigeer je naar `VillageDetailScreen` met de village `id` als parameter.

2. **VillageDetailScreen**
   - Toon afbeelding (`image_url`), naam en beschrijving.
   - Query `sights` waar `village_id` overeenkomt met het meegegeven id.
   - Haal het gemiddelde van `ratings.rating` op via een `aggregate` query.
   - Button "Geef beoordeling" opent `RatingDialog`.

3. **RatingDialog** (modal)
   - Sterrenrating (widget `RatingBar` in FlutterFlow).
   - Tekstveld voor commentaar.
   - Verstuur-knop voert een `insert` uit in `ratings` met `village_id` en `auth.user_id`.

4. **AddVillageScreen** (optioneel voor beheerders of UGC)
   - Form velden: naam, locatie (Map picker), beschrijving, foto uploaden (Supabase Storage).
   - Bij submit: insert in `villages` met `added_by = auth.user_id`.

5. **InfoScreen**
   - Bereikbaar via een info-icoon in de `AppBar`.
   - Korte uitleg over het doel van de app.

## Stap 3: Supabase integratie in FlutterFlow

- **Afbeeldingen uploaden**
  - Gebruik Supabase Storage buckets. Na upload kun je via een Custom Action de publieke URL ophalen:
  ```dart
  String getPublicUrl(String path) async {
    final response = await supabase.storage.from('images').getPublicUrl(path);
    return response.data ?? '';
  }
  ```
- **Gemiddelde rating ophalen**
  - Maak in FlutterFlow een query met `supabase.from('ratings').select('rating')` en gebruik een `aggregate` om `avg` te berekenen.
  - In custom code zou dat er zo uitzien:
  ```dart
  final response = await supabase
      .from('ratings')
      .select('rating', const FetchOptions(count: CountOption.exact))
      .eq('village_id', villageId)
      .execute();
  final avg = response.data
      .map((r) => r['rating'] as int)
      .reduce((a, b) => a + b) /
      response.data.length;
  ```
- **Markers dynamisch laden**
  - Maak een Custom Action die alle villages ophaalt en teruggeeft als lijst van `LatLng` met namen. Voeg deze lijst toe aan je `GoogleMap` markers.
- **Authenticatie**
  - Configureer Supabase auth (email/password en optioneel socials). Gebruik FlutterFlow's ingebouwde Supabase Auth widgets voor login en registratie.

## Responsiviteit en schaalbaarheid

- Gebruik `Expanded` en `Flexible` widgets zodat de lay-out op tablets en telefoons werkt.
- Maak gebruik van paginatie of lazy loading bij grote datasets.

## Toekomstige uitbreidingen (commentaar)

```dart
// TODO: notificaties bij nieuwe dorpjes in de buurt
// TODO: integratie met routeplanner API voor navigatie
// TODO: chatfunctie per dorpje
// TODO: like/report systeem bij reviews
```

Met deze stappen kun je "Hier in het dorp" volledig opzetten in FlutterFlow met Supabase als backend.
