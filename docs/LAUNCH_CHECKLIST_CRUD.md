# Launch Checklist - CRUD Funkcionalita

## 🎉 Nové funkcie

### Update 2 - Drag & Drop (18.10.2025)

Pridané drag & drop pre presúvanie úloh a kategórií!

#### ✅ Drag & Drop Features:

**1. Presúvanie úloh (zmena poradia)**
- Chyť úlohu za grip ikonu (:::)
- Presuň na inú úlohu v tej istej kategórii
- Automatická výmena `order_index` v databáze
- Vizuálny feedback: modrý ring pri drag over

**2. Presúvanie úloh medzi kategóriami**
- Chyť úlohu za grip ikonu
- Presuň na hlavičku inej kategórie
- Úloha sa presunie do novej kategórie
- Potvrdenie cez alert správu

**3. Presúvanie celých kategórií**
- Chyť kategóriu za grip ikonu v hlavičke
- Presuň na inú kategóriu
- Všetky úlohy sa presunú do cieľovej kategórie
- Confirm dialóg pred presunutím

**4. Vizuálne indikátory**
- 📌 GripVertical ikona pri každej úlohe a kategórii
- 🔵 Modrý ring pri drag over
- 🖱️ Cursor: grab → grabbing počas drag
- ⚡ Smooth transitions

---

### Update 1 - CRUD Operácie (18.10.2025)

Pridané kompletné CRUD operácie priamo v UI!

### ✅ Implementované funkcie:

#### 1. **CREATE - Pridávanie úloh**
- Tlačidlo "Pridať úlohu" v hlavnom header-i
- Tlačidlo "Pridať" pri každej kategórii (automaticky vyplní kategóriu)
- Modálne okno s formulárom:
  - Kategória
  - Úloha (popis)
  - Týždeň (1-20)
  - Poradie (automaticky vypočítané)

#### 2. **READ - Zobrazovanie úloh**
- Existujúca funkcionalita zostáva
- Kategórie s progress bar-om
- Rozbaľovanie/zabaľovanie kategórií
- Celkový pokrok

#### 3. **UPDATE - Upravovanie úloh**
- Modrá ikona ceruzky (Edit2) pri hover na úlohu
- Modálne okno s predvyplneným formulárom
- Úprava všetkých polí:
  - Kategória
  - Úloha
  - Týždeň
  - Poradie

#### 4. **DELETE - Mazanie úloh**
- Červená ikona koša (Trash2) pri hover na úlohu
- Confirm dialóg pred vymazaním
- Trvalé odstránenie z databázy

#### 5. **COPY - Kopírovanie úloh**
- Zelená ikona kópie (Copy) pri hover na úlohu
- Automaticky pridá "(kópia)" na koniec textu
- Skopíruje aj poznámky
- Nové poradie na konci zoznamu

### 🎨 UI/UX vylepšenia:

- **Hover efekty**: Akčné tlačidlá sa zobrazia len pri hover (opacity-0 → opacity-100)
- **Farebné rozlíšenie**: 
  - 🔵 Modrá = Edit
  - 🟢 Zelená = Copy
  - 🔴 Červená = Delete
- **Modálne okná**: Responzívne, tmavý overlay, animácie
- **Form validation**: Required polia, number inputs s min/max
- **User feedback**: Alert správy pri úspešných akciách

### 📋 Použitie:

#### Presunúť úlohu (zmena poradia):
```
1. Chyť úlohu za grip ikonu (:::)
2. Presuň na inú úlohu
3. Pusť - úlohy sa vymenia
```

#### Presunúť úlohu do inej kategórie:
```
1. Chyť úlohu za grip ikonu
2. Presuň na hlavičku inej kategórie
3. Pusť - úloha sa presunie
```

#### Presunúť celú kategóriu:
```
1. Chyť kategóriu za grip ikonu v hlavičke
2. Presuň na inú kategóriu
3. Potvrď - všetky úlohy sa presunú
```

#### Pridať novú úlohu:
```
1. Klikni "Pridať úlohu" (horná lišta) alebo "Pridať" (pri kategórii)
2. Vyplň formulár
3. Klikni "Pridať"
```

#### Upraviť úlohu:
```
1. Prejdi myšou na úlohu
2. Klikni modrú ceruzku
3. Uprav údaje
4. Klikni "Uložiť"
```

#### Kopírovať úlohu:
```
1. Prejdi myšou na úlohu
2. Klikni zelenú ikonu kópie
3. Kópia sa automaticky vytvorí
```

#### Vymazať úlohu:
```
1. Prejdi myšou na úlohu
2. Klikni červený kôš
3. Potvrď vymazanie
```

### 🔒 Bezpečnosť:

- RLS policies zostávajú rovnaké (admin full access)
- Confirm dialóg pri delete operácii
- Error handling s user-friendly správami
- Optimistic UI updates

### 📊 Technické detaily:

**Nové ikony z lucide-react:**
- `Plus` - Pridať úlohu
- `Edit2` - Upraviť úlohu
- `Copy` - Kopírovať úlohu
- `Trash2` - Vymazať úlohu
- `X` - Zavrieť modal
- `GripVertical` - Drag & drop handle

**Nové state variables:**
```typescript
const [showAddModal, setShowAddModal] = useState(false);
const [showEditModal, setShowEditModal] = useState(false);
const [editingItem, setEditingItem] = useState<ChecklistItem | null>(null);
const [formData, setFormData] = useState({
  category: "",
  task: "",
  week_number: 1,
  order_index: 0
});
// Drag & Drop
const [draggedItem, setDraggedItem] = useState<ChecklistItem | null>(null);
const [draggedCategory, setDraggedCategory] = useState<string | null>(null);
const [dragOverItem, setDragOverItem] = useState<string | null>(null);
```

**Nové funkcie:**
- `openAddModal(categoryName?: string)`
- `openEditModal(item: ChecklistItem)`
- `handleAddTask()`
- `handleEditTask()`
- `handleDeleteTask(itemId: string)`
- `handleCopyTask(item: ChecklistItem)`
- `handleDragStartItem(item: ChecklistItem)`
- `handleDragStartCategory(categoryName: string)`
- `handleDragOver(e: React.DragEvent, targetId: string)`
- `handleDragLeave()`
- `handleDropOnItem(targetItem: ChecklistItem)`
- `handleDropOnCategory(targetCategory: string)`

### 🐛 Known Issues / TODO:

- [x] ~~Validácia formulárov~~ ✅ Basic validation implemented
- [x] ~~Drag & drop pre zmenu poradia~~ ✅ Implemented
- [ ] Toast notifications namiesto alert()
- [ ] Batch operations (vymazať viac úloh naraz)
- [ ] History/Undo funkcionalita
- [ ] Export/Import checklist (JSON/CSV)
- [ ] Keyboard shortcuts (Ctrl+D duplicate, Del delete, etc.)

### 📈 Štatistiky:

- **Celkový počet riadkov kódu**: ~880 (z ~370 → +510)
- **Nové funkcie**: 12 (6 CRUD + 6 Drag & Drop)
- **Nové komponenty**: 2 modály
- **Nové ikony**: 6
- **Features**: CRUD + Drag & Drop

---

**Verzia**: 1.2  
**Dátum**: 18.10.2025  
**Autor**: Lectio Divina Team  
**Status**: ✅ Production Ready

### 🎯 Changelog:

**v1.2** (18.10.2025)
- ✅ Drag & drop pre úlohy (zmena poradia)
- ✅ Drag & drop úloh medzi kategóriami
- ✅ Drag & drop celých kategórií
- ✅ Vizuálne indikátory (grip ikona, ring pri hover)
- ✅ Cursor feedback (grab/grabbing)

**v1.1** (18.10.2025)
- ✅ CREATE - Pridávanie úloh
- ✅ UPDATE - Upravovanie úloh
- ✅ DELETE - Mazanie úloh
- ✅ COPY - Kopírovanie úloh
- ✅ Modálne okná s formulármi
- ✅ Hover efekty na akčných tlačidlách

**v1.0** (18.10.2025)
- ✅ Základný checklist
- ✅ Kategórie s progress bar
- ✅ Odškrtávanie úloh
- ✅ Poznámky k úlohám
