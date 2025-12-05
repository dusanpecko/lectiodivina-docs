# Nahrávanie obrázkov pre Duchovné cvičenia

## Setup Supabase Storage

### Krok 1: Vytvorenie Storage Bucketu

Spustite SQL skript v Supabase SQL Editor:

```bash
backend/sql/create_spiritual_exercises_bucket.sql
```

Alebo manuálne v Supabase Dashboard:

1. Prejdite na **Storage** → **New bucket**
2. Vyplňte údaje:
   - **Name**: `spiritual-exercises`
   - **Public bucket**: ✅ **Áno**
   - **File size limit**: 50 MB
   - **Allowed MIME types**: `image/jpeg, image/jpg, image/png, image/webp`
3. Kliknite na **Create bucket**

### Krok 2: Nastavenie Policies (automaticky v SQL)

Policies sú vytvorené automaticky SQL skriptom:
- ✅ **Public read** - každý môže čítať obrázky
- ✅ **Authenticated upload** - iba prihlásení môžu nahrávať
- ✅ **Authenticated update** - iba prihlásení môžu upravovať
- ✅ **Authenticated delete** - iba prihlásení môžu mazať

## Funkcie

### Hlavný obrázok cvičenia
- **Umiestnenie**: Admin → Duchovné cvičenia → Detail → Základné info
- **Komponent**: `ImageUploadCrop`
- **Bucket**: `spiritual-exercises`
- **Folder**: `images/`
- **Formát**: JPEG (16:9 pomer strán, 1920x1080px)
- **Kvalita**: 85%
- **Max veľkosť**: 50 MB

### Orezávanie obrázka
1. Kliknite na **"Nahrať obrázok"**
2. Vyberte obrázok z vášho počítača
3. Orezajte obrázok na požadovaný pomer strán 16:9
4. Zoomujte pomocou posuvníka
5. Kliknite na **"Uložiť obrázok"**
6. Obrázok sa automaticky nahrá do Supabase Storage

### Náhľad obrázka
- Po nahratí sa zobrazí náhľad pod tlačidlom
- Rozmer: 400x225px
- Border: svetlo sivý

### Odstránenie obrázka
- Kliknite na **"Odstrániť"** vedľa tlačidla "Nahrať obrázok"
- Obrázok sa odstráni z formulára (nie zo Storage)

## Technické detaily

### ImageUploadCrop Props
```tsx
<ImageUploadCrop
  supabase={supabase}              // Supabase client
  currentImageUrl={formData.image_url}  // Aktuálna URL
  onImageUploaded={(url) => setFormData(prev => ({ ...prev, image_url: url }))}  // Callback
  bucketName="spiritual-exercises"  // Názov bucketu
  folder="images"                   // Priečinok
/>
```

### URL štruktúra
```
https://[project].supabase.co/storage/v1/object/public/spiritual-exercises/images/1704654321000.jpg
```

### Názov súboru
- Formát: `{timestamp}.jpg`
- Príklad: `1704654321000.jpg`
- Unikátny pre každé nahratie

## Riešenie problémov

### Bucket sa nenašiel
1. Skontrolujte, či existuje bucket `spiritual-exercises` v Storage
2. Spustite SQL skript `create_spiritual_exercises_bucket.sql`

### Nedostatok oprávnení
1. Overte, že ste prihlásený používateľ
2. Skontrolujte RLS policies v Supabase Dashboard → Storage → Policies

### Obrázok sa nenačíta
1. Overte, že bucket je **public**
2. Skontrolujte URL v prehliadači
3. Vyčistite cache prehliadača

## Príklady použitia

### Vytvorenie nového cvičenia
1. Prejdite na `/admin/spiritual-exercises/new`
2. Vyplňte základné informácie
3. Kliknite na **"Nahrať obrázok"**
4. Vyberte a orezajte obrázok
5. Vytvorte cvičenie

### Úprava existujúceho cvičenia
1. Prejdite na `/admin/spiritual-exercises/[id]`
2. Ak chcete zmeniť obrázok, kliknite na **"Nahrať obrázok"**
3. Vyberte nový obrázok a orezajte ho
4. Uložte zmeny

## Budúce rozšírenia

- [ ] Galéria obrázkov (viac obrázkov pre jedno cvičenie)
- [ ] Fotka lektora (samostatný upload)
- [ ] Kompresia obrázkov na serveri
- [ ] Automatické generovanie náhľadov (thumbnails)
- [ ] Podpora WebP formátu
