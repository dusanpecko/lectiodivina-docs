# XML Import Bankových Platieb

## ✅ Funguje teraz

### Podporované formáty:
1. **CSV** - Pôvodný formát z banky
2. **XML (CAMT.053)** - Európsky štandard bankových výpisov ⭐ NOVÉ

### Ako importovať XML:

1. Prejdi na `/admin/shop/banka/import`
2. Vyber XML súbor (alebo CSV)
3. Klikni "Importovať platby"
4. ✅ Hotovo - platby sa automaticky naimportujú a vyparujú

### Čo XML parser robí:

✅ Automaticky detekuje kredit transakcie (prichádzajúce platby)  
✅ Parsuje variabilný symbol (VS) z `EndToEndId`  
✅ Extrahuje meno platiteľa  
✅ Získa IBAN platiteľa  
✅ Parsuje sumu a menu  
✅ Extrahuje dátum transakcie  
✅ Preskočí duplikáty (podľa transaction_id)  

### XML Formát (CAMT.053):

```xml
<Document>
  <BkToCstmrStmt>
    <Stmt>
      <Ntry>
        <Amt Ccy="EUR">50.00</Amt>
        <CdtDbtInd>CRDT</CdtDbtInd>
        <BookgDt><Dt>2025-11-10</Dt></BookgDt>
        <NtryDtls>
          <TxDtls>
            <Refs>
              <EndToEndId>/VS777/SS/KS</EndToEndId>
            </Refs>
            <RltdPties>
              <Dbtr><Nm>Dusan Pecko</Nm></Dbtr>
              <DbtrAcct><Id><IBAN>SK...</IBAN></Id></DbtrAcct>
            </RltdPties>
          </TxDtls>
        </NtryDtls>
      </Ntry>
    </Stmt>
  </BkToCstmrStmt>
</Document>
```

### API Endpoint:

**POST** `/api/admin/bank-payments/import-xml`

**Headers:**
```
Authorization: Bearer <admin_token>
Content-Type: multipart/form-data
```

**Body:**
```
file: <XML súbor>
```

**Response:**
```json
{
  "success": true,
  "total": 5,
  "imported": 4,
  "skipped": 1,
  "errors": 0,
  "message": "Imported 4 payments, skipped 1 duplicates, 0 errors"
}
```

### Bezpečnosť:
- ✅ Admin only (role check)
- ✅ Token authentication
- ✅ Duplicate detection
- ✅ Error handling

### Postup pri probléme:

1. **XML sa nenaimportoval?**
   - Skontroluj, či je správny formát (CAMT.053)
   - Pozri konzolu pre error message

2. **Platby sa nevyparovali?**
   - Skontroluj, či majú variabilný symbol
   - Choď na `/admin/shop/banka` a klikni "Auto-match"

3. **Duplikáty?**
   - Je to OK - systém ich automaticky preskočí
   - Transaction ID už existuje v DB

---

## 🔄 Budúcnosť: Automatický import

Plán na automatizáciu cez email: [BANK_AUTO_IMPORT_PLAN.md](./BANK_AUTO_IMPORT_PLAN.md)

- Banka pošle email na `vypis@lectio.one`
- Webhook automaticky unzipne a naimportuje
- Admin dostane notifikáciu
- **ETA: 1 deň práce** (po stabilizácii manuálneho importu)
