# Crypto Flow Graph Builder (MVP) - Dokumentacja techniczna i użytkowa

## 1. Cel i ograniczenia MVP

### 1.1 Cel
Aplikacja **Crypto Flow Graph Builder** służy do ręcznego budowania grafu przepływów środków crypto w analizie śledczej:
- obsługuje **BTC, ETH, TRON**,
- wspiera graf **cross-chain**,
- rozróżnia modele:
  - **BTC UTXO** (transakcja jako węzeł + wejścia/wyjścia),
  - **account-based** (krawędzie adres -> adres),
- zapewnia podstawową **dowodowość** (event log append-only + hash SHA-256).

### 1.2 Ograniczenia MVP
- Brak backendu i brak automatycznego pobierania danych z explorerów/API.
- Walidacja adresów/txid jest rozsądna, ale nie pełna formalnie dla wszystkich wariantów sieci.
- Sugestie klastrowania BTC są heurystyczne i nie stanowią samodzielnego dowodu.
- Praca na skali manualnej: docelowo **50-100 węzłów**.

### 1.3 Założenia uruchomienia
- Uruchomienie: dwuklik na `index.html` (Chrome/Edge).
- Brak bundlera, brak Node, brak serwera lokalnego.
- Biblioteki ładowane z CDN (wersje pinowane).

### 1.4 Użyte biblioteki (CDN, pin)
W `index.html`:
- `cytoscape@3.28.1`
- `cytoscape-edgehandles@4.0.1`
- `cytoscape-undo-redo@1.3.3`
- `cytoscape-expand-collapse@4.1.1`

---

## 2. Model danych projektu

## 2.1 Wersjonowanie
- `version` musi mieć wartość: **"1.0.0"**.
- Import projektu z inną wersją jest odrzucany walidacją.

## 2.2 Struktura główna JSON
```json
{
  "version": "1.0.0",
  "projectMeta": {
    "caseId": "...",
    "analyst": "...",
    "createdAt": "ISO-8601",
    "updatedAt": "ISO-8601",
    "notes": "..."
  },
  "elements": {
    "nodes": ["cytoscape node json"],
    "edges": ["cytoscape edge json"]
  },
  "domain": {
    "nodes": ["domena node"],
    "edges": ["domena edge"]
  },
  "eventLog": ["event entries"],
  "uiState": {
    "layout": "cose",
    "filters": {"...": "..."},
    "timeline": {"...": "..."},
    "btcSimpleView": false
  },
  "evidence": {
    "projectSha256": "hex",
    "pngSha256": "hex",
    "packageSha256": "hex"
  }
}
```

## 2.3 `projectMeta`
Pola wymagane:
- `caseId` (string)
- `analyst` (string)
- `createdAt` (ISO-8601)
- `updatedAt` (ISO-8601)
- `notes` (string)

## 2.4 `domain.nodes`
Wymagane:
- `id`, `type`, `chain`, `label`, `tags[]`, `note`, `confidence`, `createdAt`, `updatedAt`

Opcjonalne:
- `address`, `txid`, `externalId`

Dopuszczalne `type`:
- `Address`
- `Tx`
- `Entity`
- `Cluster`

Dopuszczalne `chain`:
- `BTC`, `ETH`, `TRON`

## 2.5 `domain.edges`
Wymagane:
- `id`, `type`, `chain`, `asset`, `amount`, `tags[]`, `note`, `confidence`, `createdAt`, `updatedAt`

Opcjonalne:
- `txid`, `timestamp`, `fee`, `sourceId`

Przykładowe `type`:
- `transfer` (account-based)
- `utxo-input`
- `utxo-output`
- `manual-link`
- `utxo-aggregated` (syntetyczne, tylko widok)

## 2.6 `eventLog`
Każdy wpis:
```json
{
  "ts": "ISO-8601",
  "actor": "analyst",
  "action": "ADD_NODE",
  "targetType": "node|edge|ui|projectMeta|history|mixed|...",
  "targetId": "id lub identyfikator logiczny",
  "before": {"...": "opcjonalny stan przed"},
  "after": {"...": "opcjonalny stan po"}
}
```

Zasada: log jest **append-only** podczas pracy użytkownika.

---

## 3. Teoria: UTXO vs account-based i mapowanie na graf

## 3.1 UTXO (BTC)
Model UTXO reprezentujemy jako:
- węzeł `Tx` (transakcja BTC),
- krawędzie `utxo-input`: `Address -> Tx`,
- krawędzie `utxo-output`: `Tx -> Address`.

Dzięki temu widoczna jest struktura wejść/wyjść transakcji i zależności UTXO-flow.

## 3.2 Account-based (ETH/TRON)
Model account-based reprezentujemy jako:
- krawędź `transfer`: `Address -> Address`,
- dane transferu na krawędzi: `asset`, `amount`, `txid`, `timestamp`, `fee` itd.

W MVP adresy są tworzone automatycznie, jeśli nie istnieją.

## 3.3 Widok uproszczony BTC
Tryb opcjonalny (checkbox):
- ukrywa węzły `Tx` i krawędzie `utxo-input/out`,
- generuje syntetyczne `utxo-aggregated` (`Address -> Address`).

Uwaga: to **widok przybliżony** do eksploracji i nie zastępuje pełnego modelu UTXO.

---

## 4. Definicje obiektów i atrybutów

## 4.1 Typy węzłów
- `Address`: adres blockchain.
- `Tx`: transakcja (głównie BTC UTXO).
- `Entity`: encja analityczna (np. osoba, organizacja, kontrakt, portfel zbiorczy).
- `Cluster`: klaster (compound node) grupujący ręcznie wiele węzłów.

## 4.2 Typy krawędzi
- `transfer`: transfer account-based.
- `utxo-input`, `utxo-output`: przepływy BTC względem węzła Tx.
- `manual-link`: połączenie ręczne (hipoteza analityka).
- `utxo-aggregated`: krawędź syntetyczna dla uproszczonego widoku BTC.

## 4.3 Atrybuty domenowe (rdzeń)
- `chain` - sieć (`BTC|ETH|TRON`).
- `asset` - aktywo (`BTC`, `ETH`, `TRX`, `USDT`, inne tokeny).
- `amount` - kwota (number).
- `txid` - hash transakcji.
- `timestamp` - czas zdarzenia (ISO-8601).
- `direction` - kierunek (`in|out|internal|manual` wg kontekstu).
- `fee` - opłata.
- `confidence` - wiarygodność (0..1).
- `sourceId` - źródło informacji (np. nazwa explorera/raportu).
- `note` - notatka analityczna.
- `tags[]` - etykiety/tagi.

---

## 5. Zasady cross-chain

## 5.1 Oznaczanie sieci
- Każdy węzeł/krawędź ma pole `chain`.
- UI używa kolorów i badge:
  - BTC: pomarańczowy,
  - ETH: niebieski,
  - TRON: czerwony.

## 5.2 Łączenie między sieciami
Cross-chain relacje oznaczamy jawnie przez:
- `manual-link` dla relacji analitycznych (np. bridge, CEX, swap),
- odpowiednie `tags` (`bridge`, `cex`, `swap`, `mixer`, `pośrednik`),
- `note` z kontekstem i źródłem.

## 5.3 Praktyka dowodowa
Dla relacji cross-chain zawsze wpisuj:
- skąd pochodzi wiązanie (`sourceId`),
- poziom pewności (`confidence`),
- opis logiczny w `note`.

---

## 6. Klastrowanie

## 6.1 Klastrowanie manualne
Procedura:
1. Zaznacz co najmniej 2 węzły.
2. W zakładce `Edycja` wpisz nazwę i tagi klastra.
3. Kliknij `Utwórz klaster z zaznaczonych węzłów`.
4. Użyj `Collapse/Expand` do zwijania/rozwijania.

Technicznie: klaster to compound node typu `Cluster`.

## 6.2 Sugestie heurystyczne BTC
MVP zawiera sugestię:
- **common-input ownership**: jeśli TX ma wiele inputów, można utworzyć sugerowany klaster input address.

Opcjonalne oznaczenie `change output`:
- w formularzu BTC output można podać `isChange=true`,
- wpis jest oznaczany tagiem `change-suspect`.

## 6.3 Ostrzeżenie
Heurystyki nie są dowodem samodzielnym. W UI jest ostrzeżenie, że to **"sugestia"** i wymaga niezależnej weryfikacji.

---

## 7. Dowodowość i integralność

## 7.1 Metadane sprawy
Zakładka `Dowody`:
- `caseId`
- `analyst`
- `opis/notatki`
- `data rozpoczęcia`

## 7.2 Event log append-only
Każda istotna akcja dopisuje wpis z timestampem:
- dodanie/usunięcie/edycja węzłów i krawędzi,
- klastrowanie,
- undo/redo,
- timeline/filtry,
- zapis/odczyt projektu,
- eksport PNG.

## 7.3 Hash SHA-256
Liczone są:
- hash JSON projektu (`projectSha256`),
- hash ostatnio wyeksportowanego PNG (`pngSha256`),
- hash pakietu dowodowego (`packageSha256`) liczony z:
  - `projectSha256`,
  - `pngSha256`,
  - liczby wpisów event log.

## 7.4 Odtworzenie historii
Aby odtworzyć historię:
1. Wczytaj JSON projektu.
2. Odczytaj `projectMeta` i `eventLog`.
3. Prześledź sekwencję wpisów `eventLog` (chronologicznie po `ts`).
4. Zweryfikuj hash (`projectSha256`/`packageSha256`) po ponownej serializacji.

---

## 8. Instrukcja obsługi krok-po-kroku

## 8.1 Start
1. Otwórz `index.html` w Chrome/Edge.
2. Ustaw metadane w zakładce `Dowody`.

## 8.2 Dodawanie węzłów
1. Zakładka `Dodaj` -> sekcja `Dodaj węzeł`.
2. Uzupełnij: `typ`, `chain`, `etykieta` (+ opcjonalnie `address`, `txid`, tagi, note, confidence).
3. Kliknij `Dodaj węzeł`.

## 8.3 Dodawanie transferów account-based
1. W `Dodaj transfer` podaj `fromAddress`, `toAddress`, `chain`, `asset`, `amount`.
2. Uzupełnij `txid`, `timestamp`, `fee`, `direction` opcjonalnie.
3. Kliknij `Dodaj transfer`.

## 8.4 Dodawanie transakcji BTC UTXO
1. Sekcja `Dodaj transakcję BTC (UTXO)`.
2. Podaj `txid`, `timestamp`, `fees`.
3. Wpisz `INPUTS` i `OUTPUTS` po jednej pozycji na linię.
4. Kliknij `Dodaj TX UTXO`.
5. Aplikacja tworzy: `Address -> Tx -> Address`.

## 8.5 Tworzenie klastrów
1. Zaznacz kilka węzłów.
2. `Edycja` -> `Utwórz klaster z zaznaczonych węzłów`.
3. Użyj `Collapse` lub `Expand`.

## 8.6 Sugestie klastrów BTC
1. Wybierz pojedynczy węzeł `Tx` (BTC).
2. Kliknij `Utwórz klaster z inputów wybranego TX`.
3. Sprawdź i zweryfikuj sugestię przed użyciem w raporcie.

## 8.7 Tagowanie i notatki
1. Wybierz 1 element grafu.
2. W prawym panelu edytuj pola (tagi, note, confidence, source).
3. Kliknij `Zastosuj zmiany`.

## 8.8 Filtrowanie
1. Zakładka `Filtry`.
2. Ustaw chain, asset, amount min/max, confidence min, tag filter.
3. Opcjonalnie `Ukryj edge bez timestamp`.
4. Kliknij `Zastosuj filtry`.

## 8.9 Timeline
1. Zakładka `Timeline`.
2. Ustaw zakres suwakiem lub ręcznie datami.
3. Kliknij `Zastosuj timeline`.
4. Krawędzie poza zakresem są ukrywane.

## 8.10 Undo/Redo
- `Ctrl+Z` lub przycisk `Undo`.
- `Ctrl+Y` lub przycisk `Redo`.

## 8.11 Zapis i odczyt JSON
1. Kliknij `Zapisz` lub `Zapisz projekt (JSON)`.
2. Aby odczytać: `Wczytaj` i wybierz plik `.json`.
3. Import przechodzi walidację wersji i kluczowych pól.

## 8.12 Eksport PNG
1. Ustaw skalę i tło w `Projekt/Eksport`.
2. Kliknij `Eksportuj PNG aktualnego widoku`.
3. Hash PNG pojawi się w `Dowody`.

---

## 9. Checklist testów manualnych (kryteria akceptacji)

## 9.1 Start i UI
- [ ] Dwuklik `index.html` uruchamia aplikację bez backendu.
- [ ] Zakładki i panele działają.

## 9.2 Cross-chain
- [ ] Można dodać elementy BTC + ETH + TRON w jednym grafie.
- [ ] Łańcuch jest widoczny przez kolor/oznaczenie.

## 9.3 BTC UTXO
- [ ] Dodanie BTC TX tworzy poprawnie: inputAddr -> tx -> outputAddr.
- [ ] Widok uproszczony BTC działa i można wrócić do pełnego.

## 9.4 Account-based
- [ ] Transfer ETH/TRON tworzy krawędź address->address.
- [ ] Pola `asset/amount/txid/timestamp` są zapisywane.

## 9.5 Klastrowanie
- [ ] Manualne tworzenie klastra działa.
- [ ] Collapse/Expand działa.
- [ ] Sugestia klastra z inputów BTC działa i jest oznaczona jako sugestia.

## 9.6 Undo/Redo
- [ ] Undo/Redo działa dla dodawania/usuwania/edycji elementów.
- [ ] Undo/Redo działa dla działań klastrowania.

## 9.7 Filtry i timeline
- [ ] Filtry chain/asset/amount/tag/confidence działają.
- [ ] Timeline ukrywa krawędzie poza zakresem.

## 9.8 Projekt i eksport
- [ ] Zapis JSON działa.
- [ ] Odczyt JSON z walidacją działa.
- [ ] Eksport PNG działa.

## 9.9 Dowodowość
- [ ] Event log rośnie po akcjach.
- [ ] Hash projektu i hash pakietu są wyliczane.
- [ ] Hash PNG aktualizuje się po eksporcie.

---

## 10. Plan rozwoju (architektura pod API)

## 10.1 Moduł `DataProvider`
Docelowo wydzielić warstwę:
- `DataProvider.fetchTx(chain, txid)`
- `DataProvider.fetchAddress(chain, address)`
- `DataProvider.fetchTokenTransfers(chain, address|tx)`

W MVP brak wywołań sieciowych.

## 10.2 Parsery i normalizacja
Planowane parsery:
- `BtcUtxoParser` -> normalizacja input/output,
- `EthTransferParser` -> transfery native + ERC-20,
- `TronTransferParser` -> TRX + TRC-20.

Wynik parserów mapowany do wspólnego modelu `domain.nodes/edges`.

## 10.3 Walidacja i twardy schemat
Plan:
- JSON Schema v2020-12 dla pliku projektu,
- migracje wersji (`1.0.x -> 1.1.0`),
- podpisywanie pakietów dowodowych (np. dodatkowo ECDSA).

## 10.4 Rozszerzenia analityczne
Plan:
- reguły wykrywania wzorców mixer/peel chain,
- scoring ścieżek i alerty confidence,
- porównanie wersji projektu i różnice audit trail.

---

## 11. Uwagi bezpieczeństwa i prywatności
- Aplikacja nie wysyła danych analitycznych do backendu.
- Dane pozostają po stronie użytkownika (przeglądarka + pliki lokalne).
- Ryzyko: biblioteki CDN wymagają dostępu do Internetu przy ładowaniu strony.
