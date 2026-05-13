# nokta-audit

> `@xtatistix/mobile-audit` — React Native / Expo uygulamaları için mobil bug raporlama widget'ı.

Ekran görüntüsü yakalama, sorunlu alan işaretleme, not ekleme ve Markdown / Word (.docx) formatında dışa aktarma özellikleriyle donatılmış, sürüklenebilir FAB tabanlı bir hata raporlama aracı.

---

## Özellikler

| Özellik | Açıklama |
|---|---|
| 🐛 **Sürüklenebilir FAB** | Ekranda serbestçe konumlandırılabilen, tek dokunuşla ekran yakala / çift dokunuşla not listesini aç |
| 📸 **Ekran Görüntüsü Yakalama** | Mevcut ekranın anlık görüntüsünü alır |
| 🎯 **Alan Seçimi** | Sorunlu bölgeyi parmakla çizerek sarı kutuyla işaretler, seçim ekran görüntüsüne "yanar" (burn-in) |
| 📝 **Not Ekleme** | Bottom-sheet tarzı overlay ile açıklama yazma, ekran adı ve raporlayan bilgisi otomatik eklenir |
| 📋 **Not Listesi** | Tüm notları kart görünümünde listeler — düzenleme, silme, durum takibi (açık / düzeltildi) |
| 📤 **Markdown Dışa Aktarma** | Ekran bazlı gruplanmış, durumlu, emoji destekli `.md` raporu |
| 📄 **Word Dışa Aktarma** | `docx` kütüphanesiyle profesyonel `.docx` raporu (ekran görüntüleri gömülü) |
| 💾 **Storage Soyutlaması** | `AuditStorage` interface'i üzerinden host uygulamanın istediği depolama mekanizmasını (AsyncStorage, MMKV, vb.) kullanabilir |

---

## Mimari

```
src/
├── index.ts                        # Public API — tüm export'lar
├── core/
│   ├── types.ts                    # AuditNote, AuditNoteBounds, AuditReportMeta, AuditStorage
│   └── storage.ts                  # NoteManager (CRUD), generateId
├── components/
│   ├── AuditWidget.tsx             # Ana orkestratör — FAB + state machine
│   ├── AuditSelector.tsx           # Tam ekran alan seçici (PanResponder)
│   ├── AuditOverlay.tsx            # Not yazma bottom-sheet'i
│   └── AuditNoteList.tsx           # Not listesi + düzenleme + dışa aktarma
└── export/
    ├── markdown.ts                 # buildMarkdown()
    └── docx.ts                     # buildDocx()
```

### Katmanlar

```
┌─────────────────────────────────────────────┐
│              AuditWidget (FAB)              │  ← Orkestratör
│  idle → capturing → selecting → annotating  │
│                    ↕                        │
│                   list                      │
├─────────────┬───────────────┬───────────────┤
│ AuditSelector│ AuditOverlay │ AuditNoteList │  ← UI Bileşenleri
├─────────────┴───────────────┴───────────────┤
│           NoteManager (CRUD)                │  ← İş Mantığı
├─────────────────────────────────────────────┤
│     AuditStorage (interface → host app)     │  ← Depolama Soyutlama
└─────────────────────────────────────────────┘
```

---

## Kurulum

```bash
npm install @xtatistix/mobile-audit
```

### Peer Dependency'ler

| Paket | Min. Versiyon |
|---|---|
| `react` | ≥ 18 |
| `react-native` | ≥ 0.73 |
| `expo-file-system` | ≥ 17 |
| `expo-sharing` | ≥ 11 |
| `react-native-view-shot` | ≥ 3 |

```bash
npx expo install expo-file-system expo-sharing react-native-view-shot
```

---

## Kullanım

### 1. Storage Adaptörü Oluşturma

`AuditStorage` interface'ini host uygulamanızın depolama mekanizmasıyla implemente edin:

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';
import type { AuditStorage, AuditNote } from '@xtatistix/mobile-audit';

const STORAGE_KEY = 'audit_notes';

export const auditStorage: AuditStorage = {
  async loadNotes(): Promise<AuditNote[]> {
    const raw = await AsyncStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  },
  async saveNotes(notes: AuditNote[]): Promise<void> {
    await AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(notes));
  },
};
```

### 2. Widget'ı Entegre Etme

```tsx
import React from 'react';
import { Text } from 'react-native';
import { captureScreen, captureRef } from 'react-native-view-shot';
import * as FileSystem from 'expo-file-system';
import * as Sharing from 'expo-sharing';
import { AuditWidget } from '@xtatistix/mobile-audit';
import { auditStorage } from './auditStorage';

export default function App() {
  return (
    <>
      {/* ... uygulamanızın geri kalanı ... */}

      <AuditWidget
        appName="MyApp"
        deps={{
          captureScreen: () => captureScreen({ format: 'png', result: 'tmpfile' }),
          captureRef: (ref) => captureRef(ref, { format: 'png', result: 'tmpfile' }),
          writeFile: async (filename, content) => {
            const uri = FileSystem.documentDirectory + filename;
            await FileSystem.writeAsStringAsync(uri, content);
            return uri;
          },
          writeFileBinary: async (filename, base64) => {
            const uri = FileSystem.documentDirectory + filename;
            await FileSystem.writeAsStringAsync(uri, base64, {
              encoding: FileSystem.EncodingType.Base64,
            });
            return uri;
          },
          shareFile: (uri) => Sharing.shareAsync(uri),
          storage: auditStorage,
          currentScreen: 'HomeScreen',  // dinamik olarak güncellenmeli
          reporterId: 'qa-team',        // opsiyonel
          BugIcon: <Text style={{ fontSize: 22 }}>🐛</Text>,
        }}
        initialPosition={{ bottom: 110, right: 16 }}
      />
    </>
  );
}
```

---

## API Referansı

### `AuditWidget`

Ana bileşen — ekranda sürüklenebilir FAB butonunu ve tüm alt ekranları orkestre eder.

```tsx
interface Props {
  deps: AuditWidgetDeps;
  appName?: string;                                // Varsayılan: 'App'
  initialPosition?: { bottom: number; right: number }; // FAB başlangıç konumu
}
```

#### `AuditWidgetDeps`

| Alan | Tip | Açıklama |
|---|---|---|
| `captureScreen` | `() => Promise<string>` | Ekran görüntüsü yakalar, URI döner |
| `captureRef` | `(ref) => Promise<string>` | Belirli bir View ref'ini yakalar |
| `writeFile` | `(filename, content) => Promise<string>` | Metin dosyası yazar, URI döner |
| `writeFileBinary` | `(filename, base64) => Promise<string>` | Base64 ikili dosya yazar, URI döner |
| `shareFile` | `(uri) => Promise<void>` | Dosyayı sistem paylaşım diyaloğuyla açar |
| `storage` | `AuditStorage` | Not CRUD depolama adaptörü |
| `currentScreen` | `string` | Aktif ekran adı |
| `reporterId` | `string?` | Raporlayan kişi/takım kimliği |
| `BugIcon` | `ReactNode` | FAB üzerinde gösterilen ikon |

#### Widget Durum Makinesi

```
idle ──(tek dokunuş)──→ capturing ──→ selecting ──→ annotating ──→ idle
  │                                                                  ↑
  └──(çift dokunuş)──→ list ─────────────────────────────────────────┘
```

| Durum | Davranış |
|---|---|
| `idle` | FAB görünür, sürüklenebilir |
| `capturing` | Ekran görüntüsü alınıyor |
| `selecting` | Tam ekran seçici açık — sorunlu alanı parmakla çiz |
| `annotating` | Bottom-sheet overlay — not yaz ve kaydet |
| `list` | Modal — tüm notları görüntüle, düzenle, sil, dışa aktar |

---

### Core Tipler

#### `AuditNote`

```typescript
interface AuditNote {
  id: string;
  screenName: string;
  screenshot: string;           // Dosya URI veya base64 data URI
  screenshotAspect?: number;    // Yükseklik/genişlik oranı
  highlightBounds: AuditNoteBounds | null;
  note: string;
  status: AuditNoteStatus;      // 'open' | 'fixed'
  timestamp: string;            // ISO 8601
  reporterRole?: string;
  reporterId?: string;
}
```

#### `AuditNoteBounds`

```typescript
interface AuditNoteBounds {
  x: number;
  y: number;
  width: number;
  height: number;
}
```

#### `AuditReportMeta`

```typescript
interface AuditReportMeta {
  appName: string;
  exportedAt: string;           // ISO 8601
  totalNotes: number;
}
```

#### `AuditStorage`

```typescript
interface AuditStorage {
  loadNotes(): Promise<AuditNote[]>;
  saveNotes(notes: AuditNote[]): Promise<void>;
}
```

---

### `NoteManager`

CRUD operasyonlarını yöneten sınıf. `AuditStorage` adaptörü üzerinden çalışır.

```typescript
const manager = new NoteManager(storage);

await manager.getAll();                    // Tüm notları getirir
await manager.add({ screenName, ... });    // Yeni not ekler (id, timestamp, status otomatik)
await manager.update(id, { note: '...' }); // Belirli alanları günceller
await manager.remove(id);                 // Notu siler
await manager.clear();                    // Tüm notları temizler
```

---

### Dışa Aktarma

#### `buildMarkdown(notes, meta)`

Tüm notları ekran bazlı gruplanmış Markdown formatına çevirir.

```typescript
import { buildMarkdown } from '@xtatistix/mobile-audit';

const md = buildMarkdown(notes, {
  appName: 'MyApp',
  exportedAt: new Date().toISOString(),
  totalNotes: notes.length,
});
```

**Çıktı formatı:**

```markdown
# Bug Raporu — MyApp

**Tarih:** 13.05.2026 22:30  
**Toplam:** 3 not · 🔴 2 açık · ✅ 1 düzeltildi

---

## Ekran: HomeScreen

### 🔴 #1 — Buton tıklanmıyor

![Screenshot](file:///...)

- **Durum:** Açık
- **Zaman:** 13.05.2026 22:30
- **Raporlayan:** qa-team
```

#### `buildDocx(notes, meta)`

Profesyonel Word belgesi üretir. Ekran görüntüleri base64'ten gömülü olarak eklenir, en-boy oranı korunur.

```typescript
import { buildDocx } from '@xtatistix/mobile-audit';

const base64 = await buildDocx(notes, {
  appName: 'MyApp',
  exportedAt: new Date().toISOString(),
  totalNotes: notes.length,
});
// base64 string olarak .docx içeriği — dosyaya yazılıp paylaşılabilir
```

---

## Bileşen Detayları

### `AuditSelector`

Tam ekran alan seçici. `PanResponder` kullanarak parmak hareketinden dikdörtgen seçim kutusu oluşturur.

- Ekran görüntüsü arka planda gösterilir
- Karanlık overlay ekran üzerine bindirilir
- Sarı (`#f6e05e`) kenarlıklı seçim kutusu çizilir
- "Devam →" ile seçim onaylandığında, composite view (ekran görüntüsü + sarı kutu) yakalanarak işaretlenmiş görüntü oluşturulur
- Minimum seçim boyutu: 10×10 piksel

### `AuditOverlay`

Not yazma ekranı. Bottom-sheet tarzında modal olarak açılır.

- İşaretlenmiş ekran görüntüsü küçültülmüş önizleme olarak gösterilir (maks. 160px yükseklik)
- Seçilen bölge kırmızı (`#e53e3e`) kenarlıkla vurgulanır
- Ekran adı ve raporlayan bilgisi meta olarak gösterilir
- Klavye kaçınma (KeyboardAvoidingView) desteği
- Türkçe UI: "Bug Raporu", "Sorunu açıklayın", "Kaydet", "İptal"

### `AuditNoteList`

Not listesi ve yönetim ekranı. Modal olarak tam sayfa açılır.

- Her not kart formatında: küçük ekran görüntüsü + durum rozeti + açıklama + zaman damgası
- Durum rozetleri: 🔴 Açık (kırmızı), ✅ Düzeltildi (yeşil)
- Inline düzenleme modal'ı
- Silme onay diyaloğu
- Dışa aktarma butonları: Markdown (koyu gri) ve Word (mavi)
- Boş durum: 🐛 emoji ile bilgilendirme mesajı

---

## Teknik Detaylar

| Özellik | Detay |
|---|---|
| **Dil** | TypeScript (strict mode) |
| **Hedef** | ES2020, ESNext modüller |
| **JSX** | `react-native` |
| **Modül çözümleme** | `bundler` |
| **FAB boyutu** | 52×52 piksel, kırmızı (#e53e3e), gölgeli |
| **Sürükleme eşiği** | 6 piksel (drag vs. tap ayrımı) |
| **Çift dokunuş zamanlayıcı** | 280ms |
| **Ekran kenar sınırlama** | FAB ekran dışına çıkamaz (clamp) |
| **Not ID üretimi** | `Date.now().toString(36) + Math.random().toString(36).slice(2,8)` |
| **Runtime bağımlılık** | `docx` (Word belgesi üretimi) |

---

## Proje Yapısı

```
nokta-audit/
├── .gitignore
├── package.json              # @xtatistix/mobile-audit v0.1.0
├── tsconfig.json
└── src/
    ├── index.ts              # Public barrel export
    ├── core/
    │   ├── types.ts          # 5 tip tanımı
    │   └── storage.ts        # NoteManager sınıfı + generateId
    ├── components/
    │   ├── AuditWidget.tsx   # 244 satır — FAB + state machine
    │   ├── AuditSelector.tsx # 223 satır — alan seçimi
    │   ├── AuditOverlay.tsx  # 241 satır — not yazma
    │   └── AuditNoteList.tsx # 430 satır — liste + CRUD + export
    └── export/
        ├── markdown.ts       # 45 satır — MD rapor üretici
        └── docx.ts           # 144 satır — DOCX rapor üretici
```

**Toplam:** ~1.370 satır TypeScript/TSX

---

## Lisans

MIT