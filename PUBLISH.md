# Come pubblicare questo repository

Il contenuto è già pronto. Mancano solo tre comandi.

## 1. Controlla il contenuto
```bash
cd ~/Desktop/NeuralTask-Showcase
ls -R
```
Deve esserci: `README.md`, `LICENSE`, `.gitignore`, `screenshots/` (4 immagini), `docs/ARCHITECTURE.md`.

## 2. Crea il repository su GitHub
Vai su https://github.com/new e compila così:

- **Repository name:** `NeuralTask`
- **Description:** `NeuralTask™ — the personal execution system. On-device AI for iOS. Architecture, decisions and numbers. No source code.`
- **Visibility:** ✅ **Public**
- ❌ NON spuntare "Add a README file", "Add .gitignore", "Choose a license" — li abbiamo già

⚠️ **Non chiamarlo `NeuralTask-iOS-APP`**: quel nome è già usato dal repo privato con il codice.

## 3. Collega e carica
```bash
cd ~/Desktop/NeuralTask-Showcase
git init -b main
git add .
git commit -m "NeuralTask: presentazione del prodotto, architettura e decisioni tecniche"
git remote add origin git@github.com:Alessandro-DiGiusto/NeuralTask.git
git push -u origin main
```

Se `git push` dà errore di permessi, usa HTTPS:
```bash
git remote set-url origin https://github.com/Alessandro-DiGiusto/NeuralTask.git
git push -u origin main
```

## 4. Dopo il push — due cose importanti

**a) Fissa il repo in vetrina sul tuo profilo GitHub**
Profilo → "Customize your pins" → seleziona:
1. `NeuralTask` ← questo, in prima posizione
2. `MedicaInfo-iOS-APP`
3. `Differenziata-_iOS_App`
4. `AGORA-civic-network`
5. `CRM-FullCode-NO-framework`
6. `moviafoosball-app`

Toglierai dalla vetrina i repo che ti svalutano (`Testing`, `ParallaxSite`).

**b) Aggiorna il campo "Website" del repo**
`https://neuraltask.app`

## 5. Dopo il push — collega dal sito

Nel sito nuovo aggiungi il link al repo nella sezione "03 — Prodotto" e nel footer.
