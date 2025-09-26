# Husky Watch — Account Page (MVP) Requirement Spec

**Purpose:** Implement the *minimum viable* Account Page in Flutter to support creating an **HTX** account and checking stored data.
**Scope (MVP):** Two actions only:

1. `Add Account` – create an HTX account record and (native only) store read-only API keys.
2. `Check Account` – display the account name and a **masked** Access Key read from secure storage (native) or show a prices-only notice (web).

---

## 1. Functional Requirements

### 1.1 Pages & Routes

* `AccountPage` (single page app for MVP)

  * Header: “Accounts”
  * Fields:

    * **Account Name** (TextField), default: `HTX – Personal`
    * **Access Key** (TextField) — **hidden on Web**
    * **Secret Key** (Password field with show/hide) — **hidden on Web**
  * Buttons:

    * **Add Account** → creates metadata record and saves secrets (native)
    * **Check Account** → reads back and displays Account Name + masked Access Key (native), or prices-only message (web)
  * Status output area (monospace) to show results/message

### 1.2 Supported Platforms

* **Native**: Android, iOS, macOS, Windows, Linux

  * Secrets stored via **OS keystore** (flutter_secure_storage)
* **Web (PWA)**:

  * No secret storage; key fields **hidden**
  * Display banner: “Web build runs in prices-only mode. Private keys are not saved.”

### 1.3 Data Model (Metadata only, no secrets)

```ts
Account {
  id: string      // uuid v4
  label: string   // user-defined, e.g., "HTX – Personal"
  exchange: "HTX"
  createdAt: int  // epoch ms
}
```

### 1.4 Storage

* **Account metadata** (non-secret): `SharedPreferences` (or `localStorage` on web)

  * Key: `accounts_v1` → JSON array of `Account`
* **Secrets (native only)**: `flutter_secure_storage`

  * Keys: `htx_${accountId}_ak` and `htx_${accountId}_sk`
* **No secrets on web** (throw or no-op when saving; read returns `null`)

### 1.5 Actions & Behaviors

#### Add Account

* Validate **Account Name** (non-empty; trim)
* Generate `id = uuid.v4()`
* Persist metadata into `accounts_v1` list
* **Native only**: If Access Key & Secret Key fields are non-empty → save both to secure storage under the `accountId`
* Show “Account created.” in status area and keep the newly added account as **lastAdded** (in memory)

#### Check Account

* If no `lastAdded`, show “No account yet. Add one first.”
* **Web**: show “Web build: metadata only. Private keys aren’t stored.”
* **Native**:

  * Read secrets by `accountId` from secure storage
  * If not found: “Account: <label>\nNo keys found. Please enter Access/Secret Key.”
  * If found: “Account: <label>\nAccessKey: <masked>”

    * **Masking rule**: show first 4 and last 4 characters → `"ABCD••••WXYZ"`

---

## 2. Non-Functional Requirements

* **Security (KISS)**

  * Never log secrets or canonical signature strings
  * Secrets only on native platforms via OS keystore
  * Web build: key fields hidden and storage disabled
* **Reliability**

  * All storage operations must be awaited and errors surfaced in status area
* **Performance**

  * Page loads and storage ops complete under 100ms typical on desktop
* **UX**

  * Clear inline banner on web indicating prices-only mode
  * Buttons disabled during in-flight operations (simple local bool)

---

## 3. Architecture & Code Layout

```
lib/
  core/
    accounts/
      account.dart
      account_store_prefs.dart     // SharedPreferences JSON list
    security/
      key_vault.dart               // interface
      key_vault_secure.dart        // native impl (flutter_secure_storage)
      key_vault_factory.dart       // returns NoSecretsOnWebVault or SecureKeyVault
  features/
    account/
      account_page.dart            // UI with two buttons
```

### 3.1 Interfaces

```dart
// core/accounts/account.dart
class Account {
  final String id;
  final String label;
  final String exchange; // "HTX"
  final int createdAt;
  // toJson()/fromJson()
}

// core/accounts/account_store_prefs.dart
class AccountStorePrefs {
  Future<List<Account>> getAll();
  Future<Account> add(Account a);      // append-only for MVP
}

// core/security/key_vault.dart
abstract class KeyVault {
  Future<void> save(String accountId, String accessKey, String secretKey);
  Future<({String accessKey, String secretKey})?> read(String accountId);
}

// core/security/key_vault_secure.dart
class SecureKeyVault implements KeyVault { /* uses flutter_secure_storage */ }

// core/security/key_vault_factory.dart
KeyVault createKeyVault(); // returns web or native impl
```

### 3.2 UI Contract (AccountPage)

* `Add Account` calls:

  * `AccountStorePrefs.add(Account(...))`
  * `KeyVault.save(accountId, accessKey, secretKey)` (native only)
* `Check Account` calls:

  * `KeyVault.read(accountId)` (native) → render name + masked AccessKey
  * Web → show prices-only notice

---

## 4. Dependencies

* `shared_preferences: ^2.x`
* `flutter_secure_storage: ^9.x`
* `uuid: ^4.x`

> No state management package required for MVP (use `setState`).
> Later sprints can introduce Riverpod/Bloc without changing UI contract.

---

## 5. UX Details

* **Banner (web only):** Card with italic text:

  > “Web build runs in prices-only mode. Private keys are not saved.”
* **Fields:**

  * Account Name: `TextField(hint: "HTX – Personal")`
  * Access Key (native): plain `TextField`
  * Secret Key (native): `TextField(obscureText: true, suffixIcon: visibility toggle)`
* **Buttons:**

  * `ElevatedButton("Add Account")`
  * `OutlinedButton("Check Account")`
* **Status output:** small `Text` with monospace font

---

## 6. Error Handling & Messages

* On web attempt to save secrets → show message: “Web build: private keys unsupported.”
* On secure storage failure (native) → show: “Failed to save/read keys. See logs.”
  (But do **not** include key values in logs.)
* Empty Account Name → disable Add or show inline error

---

## 7. Testing Requirements

### 7.1 Unit Tests

* `Account.fromJson/toJson()` roundtrip
* `AccountStorePrefs.add()` appends and persists correctly
* **Masking function**:

  * Input: `"ABCDEFGHIJKLMNOP"` → `"ABCD••••MNOP"`
  * Short strings (≤4) still show start then mask correctly

### 7.2 Integration (Widget) Tests

* **Native build (test using integration or platform channels mocked):**

  * Add Account with name + keys → status shows “Account created.”
  * Check Account → displays name and masked AccessKey
* **Web build (widget test with `kIsWeb` true):**

  * Key fields are **not rendered**
  * Add Account → status shows “Account created.”
  * Check Account → shows prices-only notice

### 7.3 Manual QA Checklist

* Create account with blank keys on native → Check shows “No keys found…”
* Create account with keys on native → Check shows masked AccessKey
* Logs contain **no** secrets
* Reload app → metadata remains; Check still works (native)
* Web build → never renders key fields; no exceptions thrown

---

## 8. Definition of Done (Validation Gate)

Ship only if all of the following pass:

1. **Builds** succeed for:

   * Android or iOS (one native target minimum) **and** Web
2. **Functional tests**:

   * Add Account saves metadata
   * Native: keys are stored & Check shows masked AccessKey
   * Web: Check shows prices-only message
3. **Security checks**:

   * No secrets in logs (manual grep & review)
   * Web build has **no** secret storage calls
4. **UX checks**:

   * Web banner visible
   * Buttons disabled during async ops (no double submits)
5. **Code quality**:

   * Lints clean; file structure matches spec
   * No TODOs blocking MVP behavior

---

## 9. Future Hooks (Non-blocking)

* Add **Validate** button to ping HTX `/v1/account/accounts` (signed) and show ✅/⛔
* Add **Edit/Delete**, **Export/Import** (metadata only), **Forget Secrets**
* Introduce **Riverpod** and a **Selected Account** provider
* Hook to **Asset Cards** (balances + tickers) after MVP lands

---

## 10. Handover Notes for Codex

* Implement exactly the files/classes above.
* Use `setState` for MVP; no external state mgmt.
* Guard all secret ops with `kIsWeb` checks.
* Provide small demo `main.dart` that launches `AccountPage` as `home`.
* Keep code **KISS** and production-lean: no secrets in logs, concise error messages.

**End of Spec**
