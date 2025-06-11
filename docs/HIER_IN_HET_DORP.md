# "Hier in het dorp" App Ontwerp met Supabase en FlutterFlow

Deze handleiding beschrijft stap voor stap hoe je de app *Hier in het dorp* opzet in FlutterFlow, met Supabase als backend. De app toont een kaart van Nederland met markers naar unieke dorpjes. Bij een tap krijg je details, bezienswaardigheden en gebruikersbeoordelingen.

## Stap 1: Supabase Database Schema

Voer onderstaand SQL script uit in de [SQL Editor](https://app.supabase.com/) van Supabase om de tabellen aan te maken. Dit definieert de basisstructuur van het datamodel, inclusief relaties en tijdstempels.

```sql
-- Tabel met dorpen
create table public.villages (
    id uuid primary key default gen_random_uuid(),
    name text not null,
    location_lat float8 not null,
    location_lng float8 not null,
    description text,
    image_url text,
    added_by uuid references auth.users not null,
    created_at timestamp with time zone default timezone('utc', now()) not null
);

-- Bezienswaardigheden per dorp
create table public.sights (
    id uuid primary key default gen_random_uuid(),
    village_id uuid references public.villages on delete cascade not null,
    title text not null,
    description text,
    image_url text,
    created_at timestamp with time zone default timezone('utc', now()) not null
);

-- Beoordelingen per dorp
create table public.ratings (
    id uuid primary key default gen_random_uuid(),
    village_id uuid references public.villages on delete cascade not null,
    user_id uuid references auth.users not null,
    rating int check (rating between 1 and 5) not null,
    comment text,
    created_at timestamp with time zone default timezone('utc', now()) not null
);

-- Favoriete dorpen (toekomstige functionaliteit)
create table public.favorites (
    id uuid primary key default gen_random_uuid(),
    village_id uuid references public.villages on delete cascade not null,
    user_id uuid references auth.users not null,
    created_at timestamp with time zone default timezone('utc', now()) not null,
    unique (village_id, user_id)
);
```

Zorg ervoor dat [Row Level Security (RLS)](https://supabase.com/docs/guides/auth/row-level-security) is ingeschakeld en schrijf policies die alleen geauthenticeerde gebruikers toestaan om eigen ratings/favorieten aan te maken. Voor `villages` kun je optioneel restricties toevoegen zodat alleen admins dorpen mogen toevoegen.

### Storage buckets

Maak twee buckets aan in Supabase Storage:
- `village-images`
- `sight-images`

Beide buckets kunnen publiek leesbaar worden ingesteld zodat de afbeeldingen direct geladen kunnen worden in de app.

## Stap 2: FlutterFlow Project Structuur

1. **Start een nieuw FlutterFlow project** met Supabase als backend. Configureer in *Settings → Integrations → Supabase* de URL en anon/public key van je Supabase project.
2. Maak de volgende schermen aan in het *UI Builder*:
   - `HomeScreen`
   - `VillageDetailScreen`
   - `AddVillageScreen` (optioneel voor beheerders)
   - `RatingDialog` (modal widget)
   - `InfoScreen` (bereikbaar via een info-icoon in de AppBar)
3. Voeg in het project de benodigde custom actions toe (zie verderop) en stel in *Authentication* het login- en registratieproces in met Supabase auth.

## Stap 3: Schermspecificaties

### 1. HomeScreen

- Gebruik de **Google Maps** widget ("GoogleMap") als hoofdcomponent.
- Haal via een Supabase query alle `villages` op.
- Maak dynamisch markers aan voor elke village. Dit kan via een custom action die door de query iterates en `Marker` widgets toevoegt.

Pseudo-code voor het plaatsen van markers in een custom action:

```dart
Future<List<Marker>> generateVillageMarkers(List villages) async {
  return villages.map((v) => Marker(
    markerId: MarkerId(v['id'] as String),
    position: LatLng(v['location_lat'] as double, v['location_lng'] as double),
    onTap: () {
      context.pushNamed('VillageDetailScreen', params: {'villageId': v['id']});
    },
  )).toList();
}
```

- Koppel de lijst met markers aan de `initialMarkers` property van de Google Map.
- Plaats in de AppBar een info-icoon (`Icons.info_outline`) dat naar `InfoScreen` navigeert.
- Voeg een *floating action button* toe (alleen zichtbaar voor admins) om `AddVillageScreen` te openen.

### 2. VillageDetailScreen

Dit scherm toont uitgebreide info over het gekozen dorp.

- Query `villages` met het `villageId` dat vanuit `HomeScreen` is meegegeven. Gebruik `SingleRecord Query` om één record op te halen.
- Toon de `image_url` via een `CachedNetworkImage` widget en plaats naam en beschrijving in een `Column`.
- Query `sights` met filter `village_id = villageId` en toon de resultaten in een `ListView` met cards en afbeeldingen.
- Voor de gemiddelde rating kun je Supabase's `rpc` functie gebruiken of de `select().avg('rating')` aggregate query. In FlutterFlow kun je dit instellen via een `Custom Query`.

Voorbeeld van aggregate query (Supabase JS/Flutter syntax):
```dart
final avgResponse = await supabase
  .from('ratings')
  .select('rating', const FetchOptions(head: true, count: CountOption.exact))
  .eq('village_id', villageId)
  .avg('rating');
```

- Toon het gemiddelde in sterren (bijv. met `SmoothStarRating` widget) en het aantal reviews.
- Plaats een button "Geef beoordeling" die `showModalBottomSheet` of een dialog opent (`RatingDialog`).

### 3. RatingDialog

- Bestaat uit een sterrenrating (`RatingBar`), een tekstveld voor commentaar en een "Submit" knop.
- Bij submit wordt een record toegevoegd aan `ratings` via `insert`.

FlutterFlow Custom Action voorbeeld (simplified):
```dart
Future<void> submitRating(double rating, String comment, String villageId) async {
  final userId = supabase.auth.currentUser!.id;
  await supabase.from('ratings').insert({
    'village_id': villageId,
    'user_id': userId,
    'rating': rating.round(),
    'comment': comment,
  });
}
```
- Na het opslaan kun je de dialog sluiten en de lijst met ratings verversen.

### 4. AddVillageScreen (optioneel)

Alleen toegankelijk voor beheerders of trusted users.

- Form fields: naam, beschrijving, kaart-picker voor locatie (lat/lng), upload widgets voor afbeelding.
- Afbeelding wordt geüpload naar Supabase Storage `village-images` bucket. Gebruik een Custom Action om de upload URL te verkrijgen en op te slaan in `image_url`.

Upload snippet (vereenvoudigd):
```dart
Future<String> uploadVillageImage(File file) async {
  final fileExt = file.path.split('.').last;
  final fileName = '${Uuid().v4()}.$fileExt';
  final storageResponse = await supabase.storage
      .from('village-images')
      .upload(fileName, file);
  if (storageResponse.error != null) {
    throw storageResponse.error!;
  }
  final publicUrl = supabase.storage
      .from('village-images')
      .getPublicUrl(fileName);
  return publicUrl;
}
```

- Bij het submitten van het formulier wordt een nieuw record in `villages` aangemaakt met het resultaat van deze upload functie in `image_url` en de huidige gebruiker in `added_by`.

### 5. InfoScreen

Simpel scherm met uitleg over de app, hoe je dorpen kunt ontdekken en beoordelingen kunt achterlaten.

## Stap 4: Extra Technische Details

- **Loading-indicatoren**: gebruik `CircularProgressIndicator` tijdens Supabase queries.
- **Responsive design**: gebruik `ResponsiveVisibility` widgets en zorg voor flexibele layout (bijv. `Expanded` in `Row`/`Column`). Test zowel mobiel als tablet in de FlutterFlow builder.
- **Favorites**: De tabel `favorites` is al aangemaakt. Voeg later een bookmark-icoon toe op `VillageDetailScreen` die een record aanmaakt of verwijdert in deze tabel.
- **RLS policies**: minimaal `authenticated` check voor `insert` op `ratings` en `favorites`. Voorbeeld:

```sql
create policy "Allow insert own ratings" on public.ratings
  for insert with check (auth.uid() = user_id);
```

- Voeg in commentaar eventueel toekomstige uitbreidingen toe, zoals notificaties bij nieuwe dorpen in de buurt, routeplanner integratie, in-app chat en like/report systeem voor reviews.

## Voorbeeld van uitbreiding (commentaar)

```dart
// TODO: Push notificaties bij nieuwe dorpjes in de buurt
// TODO: Integratie met externe routeplanner API
// TODO: Chat per dorpje met Supabase Realtime
// TODO: Mogelijkheid om reviews te liken of te rapporteren
```

Met deze handleiding kun je de app *Hier in het dorp* opzetten met FlutterFlow en Supabase. De structuur zorgt ervoor dat de app schaalbaar, responsive en uitbreidbaar blijft.

