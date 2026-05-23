# Sosty - Gestione Sostituzioni Giornaliere Docenti

Sosty è un'applicazione web open-source progettata specificamente per le segreterie e i team digitali delle istituzioni scolastiche. Permette di automatizzare, pianificare e comunicare in modo efficiente le sostituzioni giornaliere dei docenti assenti.

L'applicazione è ottimizzata per essere eseguita **localmente su Ubuntu Server** all'interno della rete scolastica, garantendo la massima velocità di caricamento e la piena conformità al GDPR (nessun dato personale dei docenti esce dall'istituto).

---

## 🚀 Funzionalità Principali

* **Algoritmo di Sostituzione Automatica:** Propone i sostituti ideali incrociando l'orario d'istituto in base a una gerarchia blindata (Docenti di sostegno compresenti ➡️ Docenti a disposizione/recupero ➡️ Assistenti).
* **Gestione Permessi Multi-Ora:** Supporta l'inserimento sia dell'intera giornata di assenza che di permessi brevi su blocchi di ore specifici (es. 1,2,3 ora).
* **Pianificazione Futura:** Consente di navigare sul calendario per pianificare le supplenze dei giorni successivi, bloccando automaticamente i fine settimana per evitare errori.
* **Monitor Totem per i Corridoi:** Include una rotta pubblica (`/totem-corridoio`) pensata per i display della scuola, con carosello automatico delle variazioni d'orario, bacheca avvisi interni e orologio live.
* **Generatore di Circolari Rapido:** Crea istantaneamente il testo formattato con ore in numeri romani e note cattedra, pronto da copiare e incollare su bacheche d'istituto o canali di comunicazione.
* **Esportazione Log:** Download dei registri delle sostituzioni in formato `.csv` sia giornaliero che mensile per lo storico amministrativo.

---

## 🛠️ Requisiti d'Installazione (Server di Scuola)

* Ubuntu Server (o qualsiasi macchina Linux/macOS/Windows con supporto Docker)
* Docker e Docker Compose

---

## 📦 Avvio Rapido con Docker

1. Clona questo repository sul server della scuola:
   ```bash
   git clone [https://github.com/tuo-username/sostituzioni-facili.git](https://github.com/tuo-username/sostituzioni-facili.git)
   cd sostituzioni-facili
