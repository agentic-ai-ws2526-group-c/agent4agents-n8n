# Agent4Agents - n8n Framework Recommender

Dieses Repository enthält einen n8n-Workflow im JSON-Format, der als intelligentes Beratungstool zur Auswahl von **Agentic-AI-Frameworks** dient. Das System nutzt ein "Dual-Path"-Verfahren: Es kann entweder über ein **Formular** für Endnutzer oder als **Evaluations-Workflow** zur Qualitätssicherung betrieben werden.

## 📁 Dateistruktur

* `agent4agents-n8n.json`: Die vollständige Workflow-Datei für den Import in n8n.
* `README.md`: Dokumentation und Installationsanleitung.
* `agent_eval_n8n`: Vorlage für das Evaluations Google Sheet


---

## 🔧 Installation & Setup

### 1. Workflow importieren
1. Man lädt die Datei `agent4agents-n8n.json` aus diesem Repository herunter.
2. In n8n klickt man oben rechts auf das **Menü** und wählt **"Import from File"**.
3. Man wählt die JSON-Datei aus.


### 2. Zugangsdaten (Credentials) konfigurieren
Nach dem Import müssen die eigenen API-Schlüssel in den entsprechenden Nodes hinterlegt werden:
* **Google Gemini (PaLM) API:** Erforderlich für die Agenten-Nodes.
* **Google Service Account API:** Erforderlich für den Datenzugriff auf das Google Sheet.
* **Tavily API:** Für die Suchfunktion der Agenten.
* **Context7:** Für den Zugriff auf Context7. Im `MCP Client Context7` Node müssen die Credentials wie folgt abgespeichert werden:

#### Credential for Header Auth
| Eigenschaft | Wert |
| :--- | :--- |
| **Endpoint** | mcp.context7.com/mcp |
| **Server Transport** | HTTP Streamable |
| **Authentication** | Header Auth |
| **Name** | Authorization |
| **Value** | Bearer DEIN_CONTEXT7_API_KEY |



---

## 🛠 Die zwei Betriebsmodi

Der Workflow erkennt automatisch, welcher Pfad genutzt wird, und verhält sich entsprechend:

### 1. Der Formular-Modus (Live-Betrieb)
* **Trigger:** `On form submission` (n8n Form Node).
* **Ablauf:** Ein Nutzer füllt ein bereitgestelltes Formular aus.
* **Ergebnis:** Man erhält sofort ein visuelles **HTML-Dashboard** mit der Framework-Empfehlung, Begründung und einer Qualitätsbewertung durch einen zweiten Agenten ("Judge Agent").

### 2. Der Evaluations-Modus (Batch-Test)
* **Trigger:** `When fetching a dataset row` (Google Sheets). Diese sind momentan im Node auf 16 rows begrenzt und muss bei mehr UseCases überarbeitet werden (Max Rows to Process).
* **Ablauf:** Der Workflow liest vordefinierte Test-Szenarien aus einer Google-Tabelle. Die Agenten bearbeiten die Anfrage und die Correctness (0 = stimmt mit Google Sheet Lösung nicht überein, 1 = stimmt überein) wird geprüft.
* **Ergebnis:** Die KI-Empfehlungen und Scores werden direkt in die Google-Tabelle zurückgeschrieben.


---

## ⚙️ Betrieb des Workflows

### 📝 Verwendung des Formulars
Man öffnet den Node `On form submission` und nutzt die bereitgestellte **Test URL** (auch Production URL möglich). Das Absenden des Formulars startet die Analyse; das Ergebnis wird direkt im Browser-Interface gerendert. Das Ergebnis ist an das Formular des Google ADK Prototypen angepasst, hier sind allerdings nur Dummy-Buttons erstellt.

### 📊 Verwendung des Evaluations-Modus
1. Man verknüpft ein Google Sheet mit den erforderlichen Headern (hier am Besten die Vorlage verwenden).
2. Man startet den Node `When fetching a dataset row`.
3. Der Workflow verarbeitet die Daten und die Antworten der Agenten und überträgt die Evaluierung in die Tabelle.

---

## 🤖 Aufbau und Logik der Agenten

Die Agentenlogik unterteilt sich in zwei sequenziel laufende Agenten:

### RecommenderAgent (Der Strategie-Experte)
Dieser Agent nutzt den **COMPASS-System-Prompt**. Er ist darauf programmiert, Anforderungen (Usecase, Interaktionskanal, Integrationen) aus einem Formular oder der Evaluationstabelle zu analysieren und das effizienteste Framework auszuwählen. 
* **Entscheidungslogik:** Er priorisiert "Ease of Use". Nur wenn ein Usecase eine Komplexität aufweist, die Low-Code-Tools (wie n8n oder Cognigy) übersteigt, empfiehlt er High-Code-Frameworks (wie LangGraph oder CrewAI).
* **Tools:** Der Agent kann an die Tools **Tavily Search** (für aktuelle Framework-Recherchen) und den **Context7 MCP Client** angebunden werden. Dafür muss man die Tool nodes an den Reiter Tools des Agenten anbinden.

### 2 JudgeAgent (Die Qualitätsinstanz)
Dieser Agent fungiert als unabhängiger Reviewer. Er erhält sowohl die ursprüngliche Nutzeranfrage als auch die Antwort des RecommenderAgenten.
* **Bewertung:** Er vergibt einen Score von 1 bis 10.
* **Kriterien:** Er prüft auf technologische Konsistenz, realistische Einschätzung des Implementierungsaufwands und mögliche Fehlentscheidungen des ersten Agenten.

### Structured Output Nodes (Daten-Parsing)
Obwohl in den System-Prompts bereits ein JSON-Format vorgegeben ist, nutzt der Workflow dedizierte **Structured Output Parser Nodes**. Diese sind aus folgenden Gründen essenziell für die Stabilität:
* **Technische Schnittstelle:** Die Parser wandeln den reinen Text-Output der KI in echte n8n-Datenobjekte um. Ohne diese Nodes könnten nachfolgende Nodes (wie das Google Sheet oder das HTML-Formular) nicht gezielt auf einzelne Felder wie `framework` oder `score` zugreifen.
* **Format-Validierung:** Sie stellen sicher, dass die KI das geforderte Schema exakt einhält. Sollte die KI ungültiges JSON liefern, erzwingt der Parser eine Korrektur, bevor die Daten den Workflow weiter durchlaufen.
