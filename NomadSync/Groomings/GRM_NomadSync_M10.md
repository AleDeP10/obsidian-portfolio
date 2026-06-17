# GRM — Milestone 9: MainWindow (JavaFX), PrismUI, i18n

---

## Contesto

La milestone 5 ha rilasciato ObsidianSync 1.0.0 con tray funzionante e backend completo. La milestone 6 introduce `MainWindow` tramite **JavaFX**, il design system condiviso (`PrismUI`) e l'internazionalizzazione su 10 lingue.

La coesistenza AWT (tray) + JavaFX (MainWindow) è supportata: `Platform.setImplicitExit(false)` — la JVM non esce quando la finestra viene chiusa. Tutti gli aggiornamenti JavaFX da thread AWT usano `Platform.runLater()`.

**Prerequisito operativo**: completare il tutorial JavaFX (scene graph, FXML, CSS, `Platform.runLater()`, coesistenza AWT) prima di toccare qualsiasi componente UI — come concordato nelle metriche di apprendimento.

---

## Obiettivi

|#|Obiettivo|Criterio di accettazione|
|---|---|---|
|1|Tutorial JavaFX completato|scene graph, FXML, CSS, threading, coesistenza AWT compresi|
|2|`MainWindow` — struttura|finestra apre da ContextMenu, vault switcher funzionante|
|3|Tab Home|dashboard con card per vault, alert bar conflitti|
|4|Tab Properties|form editing vault, validazione, salvataggio|
|5|Tab Log|visualizzatore eventi, filtro livello e date, auto-scroll|
|6|Tab Conflicts|lista snapshot, apertura cartella, dialog conferma risoluzione|
|7|Tab Backup|lista snapshot FIFO, ripristino, eliminazione|
|8|Tab Settings|configurazione globale: porta, lingua, tema|
|9|`PrismUI` — progetto Maven separato|tema Default funzionante, dipendenza inclusa|
|10|Temi UI|Default + Retro terminal + Zen minimal via CSS swap|
|11|i18n — 10 lingue|`ResourceBundle` configurato, stringhe esternalizzate|
|12|Installer Windows|`.exe` via `jpackage`, shortcut desktop, avvio automatico|
|13|Pubblicazione `PrismUI` su Maven Central|artefatto pubblicato, README con screenshot|
|14|Release 2.0.0|`git flow release finish 2.0.0`, tag pushato|

---

## Ordine implementativo

### Layer 0 — Prerequisiti (bloccante)

**Step 0 — Tutorial JavaFX** Argomenti obbligatori prima di scrivere una riga di UI:

- Scene graph: `Stage` → `Scene` → `Parent` → nodi
- FXML + `FXMLLoader` — separazione layout/logica
- CSS JavaFX — selettori, variabili, tema switching
- `Platform.runLater()` — aggiornamenti UI da thread non-JavaFX
- Coesistenza AWT/JavaFX: `Platform.setImplicitExit(false)`

Dipendenze Maven:

```xml
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-controls</artifactId>
    <version>21</version>
</dependency>
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-fxml</artifactId>
    <version>21</version>
</dependency>
```

---

### Layer 1 — PrismUI design system

**Step 1 — Progetto Maven `prism-ui`**

```
prism-ui/
  src/main/java/io/aledep10/prismui/
    components/     ← PButton, PLabel, PTextField, PComboBox, PTabPane
    theme/          ← ThemeManager
  src/main/resources/io/aledep10/prismui/themes/
    default.css
    retro-terminal.css
    zen-minimal.css
```

Tema switching:

```java
scene.getStylesheets().clear();
scene.getStylesheets().add(getClass().getResource("themes/" + theme.cssFile()).toExternalForm());
```

Naming session per il nome definitivo prima del release — candidato principale **PrismUI**.

---

### Layer 2 — MainWindow

**Step 2 — Struttura finestra**

`Stage` con `BorderPane`. Vault switcher (`ComboBox` autocomplete) in toolbar superiore, sempre visibile. Sei tab:

```
[ 🏠 Home ]  [ Properties ]  [ Log ]  [ Conflicts ]  [ Backup ]  [ ⚙ Settings ]
```

Apertura contestuale da ContextMenu AWT (thread-safe):

```java
Platform.runLater(() -> mainWindow.openTab(Tab.LOG, vaultId));
Platform.runLater(() -> mainWindow.openTab(Tab.CONFLICTS, vaultId));
```

Default (da TrayIcon / eseguibile): tab Home.

**Step 3 — Tab Home** Card per vault: nome, stato icona, ultimo sync, pulsante Sync. Alert bar arancione se conflitti pending — link diretto a tab Conflicts.

**Step 4 — Tab Properties** Form FXML per configurazione per-vault: nome, path, remote URL, branch, credenziali. Validazione inline con `TextFormatter`. Salvataggio via `VaultService.update()`.

**Step 5 — Tab Log** `TextArea` non editabile. `ChoiceBox` filtro livello, `DatePicker` filtro date, toggle auto-scroll, pulsante Clear. Aggiornamento asincrono via `Platform.runLater()`.

**Step 6 — Tab Conflicts** `ListView` di snapshot in `remote-conflicts/<vault>/`. Per ogni snapshot: timestamp, lista file, pulsante "Apri cartella" → `Desktop.getDesktop().open()`.

Dialog di conferma risoluzione:

```
Hai copiato le modifiche remote che volevi preservare?

Una volta confermato, la versione remota di
note-lavoro.md verrà eliminata.

[ Annulla ]    [ Sì, ho finito ]
```

**Step 7 — Tab Backup** `ListView` di snapshot FIFO in `backups/<vault>/`. Per ogni snapshot: timestamp, dimensione, pulsante "Apri cartella", pulsante "Ripristina", pulsante "Elimina".

**Step 8 — Tab Settings** Porta socket, intervallo autosave, percorso backup, soglia FIFO, tema UI (cambio live), lingua.

---

### Layer 3 — i18n

**Step 9 — Esternalizzazione stringhe** `ResourceBundle.getBundle("messages", locale)`. Cambio lingua → reload + aggiornamento binding.

|Lingua|Codice|
|---|---|
|English|`en`|
|Mandarin|`zh`|
|Hindi|`hi`|
|Spanish|`es`|
|Arabic|`ar`|
|Portuguese|`pt`|
|French|`fr`|
|German|`de`|
|Japanese|`ja`|
|Italian|`it`|

RTL: `root.setNodeOrientation(NodeOrientation.RIGHT_TO_LEFT)` — propaga ai figli automaticamente.

---

### Layer 4 — Installer e release

**Step 10 — Installer Windows** `jpackage` → `.exe` con JRE bundled, shortcut desktop, avvio automatico, uninstaller.

**Step 11 — Pubblicazione PrismUI su Maven Central** Signing GPG + deploy su OSSRH. README con screenshot dei tre temi. Avviare registrazione OSSRH almeno 2 settimane prima del release.

**Step 12 — Release 2.0.0**

```
git flow release start 2.0.0
git flow release finish 2.0.0
git push origin main develop --tags
```

---

## Rischi

|Rischio|Probabilità|Impatto|Mitigazione|
|---|---|---|---|
|Coesistenza AWT + JavaFX — threading issues|Media|Alto|`Platform.setImplicitExit(false)`, update via `Platform.runLater()`|
|JavaFX rendering inconsistente Windows 10 vs 11|Bassa|Medio|test su entrambe le versioni|
|`jpackage` — installer ~60MB con JRE bundled|Alta|Basso|accettato — standard JavaFX desktop|
|Traduzione Mandarin/Arabo — revisione madrelingua|Alta|Medio|DeepL per bozza, flag "community review needed"|
|Maven Central — approvazione 1-2 settimane|Alta|Basso|avviare registrazione OSSRH in anticipo|
|RTL layout rompe componenti non preparati|Media|Medio|testare con locale `ar` in sviluppo|

---

## Scope rimandato a M7

- React Native mobile (iOS / Android / Huawei AppGallery)
- Temi aggiuntivi — v1.1