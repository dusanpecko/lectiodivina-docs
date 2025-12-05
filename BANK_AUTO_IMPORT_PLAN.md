# Automatický Email Import - Plán implementácie

## Fáza 1: ✅ Manuálny XML import (HOTOVO)
- [x] XML parser pre CAMT.053 formát
- [x] API endpoint `/api/admin/bank-payments/import-xml`
- [x] Frontend podpora pre upload XML súborov
- [x] Automatické parsovanie VS, sumy, platiteľa

## Fáza 2: 🔄 Automatický email import (BUDÚCNOSŤ)

### Technológie a služby:
1. **Email prijímanie:**
   - Gmail API s custom doménou (vypis@lectio.one)
   - Amazon SES Email Receiving
   - Microsoft Graph API (ak máš Microsoft 365)
   - SendGrid Inbound Parse

2. **Odporúčanie: Gmail API + Cloud Function**
   ```typescript
   // Pushnúť do Google Cloud Function
   // Trigger: Email na vypis@lectio.one
   ```

### Postup implementácie:

#### 1. Email Handler Setup
```typescript
// /api/webhooks/bank-statement/route.ts
export async function POST(request: NextRequest) {
  // 1. Prijmi email webhook
  const emailData = await request.json();
  
  // 2. Kontrola odosielateľa (whitelist)
  if (!isValidSender(emailData.from)) {
    return NextResponse.json({ error: 'Unauthorized sender' }, { status: 403 });
  }
  
  // 3. Stiahni attachment
  const attachment = emailData.attachments[0];
  
  // 4. Unzip protected file
  const unzipped = await unzipFile(attachment.data, 'Neberemhackeru2026*');
  
  // 5. Parse XML
  const payments = await parseCAMT053(unzipped);
  
  // 6. Import do DB
  await importPayments(payments);
  
  // 7. Automaticky vyparuj
  await autoMatch();
  
  // 8. Pošli notifikáciu adminovi
  await sendNotification({
    subject: 'Bankový výpis importovaný',
    imported: payments.length
  });
  
  return NextResponse.json({ success: true });
}
```

#### 2. Unzip funkcia
```typescript
import AdmZip from 'adm-zip';

async function unzipFile(zipBuffer: Buffer, password: string): Promise<Buffer> {
  const zip = new AdmZip(zipBuffer);
  const entries = zip.getEntries();
  
  for (const entry of entries) {
    if (entry.name.endsWith('.xml')) {
      // Decrypt with password
      const decrypted = zip.readFile(entry, password);
      return decrypted;
    }
  }
  
  throw new Error('No XML file found in ZIP');
}
```

#### 3. Email Forwardovanie
Nastaviť v banke:
- Emailová adresa: `vypis@lectio.one`
- Forward na webhook URL: `https://lectio.one/api/webhooks/bank-statement`

Alebo použiť Gmail:
1. Vytvoriť `vypis@lectio.one` v Google Workspace
2. Nastaviť Gmail API filter
3. Webhook na každý nový email

### Bezpečnosť:
- ✅ Whitelist emailových adries banky
- ✅ Password protection na ZIP
- ✅ Webhook secret token
- ✅ Rate limiting
- ✅ Notifikácie pri chybách

### Monitoring:
- Log každého importu
- Alert pri zlyhaniach
- Denný report (koľko platieb, vyparovaných, atď.)

### Potrebné balíčky:
```bash
npm install adm-zip
npm install @google-cloud/functions-framework
npm install @sendgrid/inbound-mail-parser
```

### Cena služieb:
- Gmail API: Free (do 1M requests/deň)
- Cloud Function: ~$0 (free tier 2M invocations)
- SendGrid Inbound: Free

### Časový odhad implementácie:
- Email setup: 2 hodiny
- Webhook handler: 3 hodiny
- Unzip + parser: 2 hodiny
- Testing: 2 hodiny
- **Celkom: ~1 deň práce**

## Migračný plán:

### Teraz (Manuálny):
1. Admin dostane email s výpisom
2. Stiahne ZIP
3. Unzipne (heslo: Neberemhackeru2026*)
4. Nahráa XML na `/admin/shop/banka/import`
5. Automatické parsovanie a import
6. Automatické vyparovanie
7. Manuálne doriešenie nespárovaných

### Budúcnosť (Automatický):
1. Banka pošle email na `vypis@lectio.one`
2. Webhook prijme email
3. Automatický unzip
4. Automatický import
5. Automatické vyparovanie
6. Admin dostane report email
7. Admin len skontroluje nespárované

### Výhody automatizácie:
✅ Žiadna manuálna práca
✅ Real-time import (hneď po príchode emailu)
✅ Eliminácia chýb
✅ História a audit trail
✅ Instant notifikácie

### Kedy implementovať?
**Odporúčam:** Po 2-3 mesiacoch používania manuálneho importu
- Overíš, že všetko funguje správne
- Zistíš edge cases
- Vieš presne, čo automatizovať

---

## Aktuálny stav:
✅ **XML import funguje manuálne**
✅ **Parsovanie CAMT.053 hotové**
✅ **Auto-matching implementovaný**
🔄 **Automatický email import - pripravený plán**
