
# Sicherheitsüberprüfung Accent GeoGuessr

## Status der Anwendung

Die öffentlich erreichbare Instanz der Anwendung unter accentgeoguessr-classroom.streamlit.app wurde momentan vorübergehend abgeschaltet. Dies ermöglicht die Umsetzung der in diesem Bericht beschriebenen Maßnahmen, ohne dass während der Bereinigung ein aktives Risiko für Nutzer besteht. Die Untersuchungsergebnisse beziehen sich auf den Stand vor der Abschaltung. Sobald die Korrekturen umgesetzt sind, kann die Anwendung mit aktualisierter Sicherheit wieder veröffentlicht werden.

## Ziel und Umfang der Untersuchung

Untersucht wurde die Anwendung aus dem öffentlichen GitHub Repository github.com/quentinrauschenbach/accent_geoguessr. Der gesamte im Repository enthaltene Code wurde durchgesehen, mit besonderem Fokus auf die beiden Hauptdateien app.py und app_local.py, da sie die komplette Anwendungslogik enthalten. Zusätzlich wurden die Abhängigkeiten in requirements.txt und die Commit Historie betrachtet. Ergänzend wurde die live betriebene Instanz manuell besucht und geprüft, und zwar beide Bereiche der Anwendung: die normale Schüleransicht unter https://accentgeoguessr-classroom.streamlit.app/ sowie das Lehrerpanel unter https://accentgeoguessr-classroom.streamlit.app/?role=teacher. Die Prüfung umfasste die Serverantworten, die gesetzten Cookies und die sichtbare Konfiguration beider Bereiche.

## 1. Zusammenfassung

Die Anwendung legt den gesamten Spielzustand in einem globalen Speicher auf dem Server ab, kennt praktisch nur für die Lehrkraft eine Authentifizierung, und diese ist schwach umgesetzt. Für eine reine Spielerei im Klassenzimmer wären die meisten Punkte verkraftbar. Die Instanz lief jedoch öffentlich im Internet, und dadurch sind die kritischen Funde real ausnutzbar. Das Hauptrisiko liegt nicht in klassischen Injektionsschwachstellen, sondern in Logik und Konfiguration: Übernahme der Lehrerrolle, Manipulation des Spielstands und Phishing der Schüler über einen manipulierten QR Code. Ergänzend kommen Schwächen auf der Auslieferungsebene hinzu, die beim Betrachten der Serverantworten aufgefallen sind.

## 2. Kritische Funde

### 2.1 Passwort im Klartext im öffentlichen Repository

In app.py ist das Lehrerpasswort Zeile für Zeile lesbar hinterlegt:

```python
TEACHER_PASSWORD = "0712"
```

Das Passwort ist öffentlich einsehbar. Wer den Repositorynamen kennt, kann die Lehrerfunktion sofort übernehmen. Zusätzlich handelt es sich um vier Ziffern, die sehr nach einem Datum aussehen, also auch bei einer Änderung leicht erratbar sind. Ein bloßes Ändern der Datei genügt nicht, weil der alte Wert in der Commit Historie dauerhaft erhalten bleibt. Zur Bereinigung der Historie eignen sich Werkzeuge wie BFG Repo Cleaner oder git filter repo. Im einfachsten Fall gilt der alte Wert als endgültig verworfen und wird nie wieder verwendet.

Empfehlung: Das Passwort in Streamlit Secrets auslagern, also über st.secrets laden. Die Datei secrets.toml gehört in die gitignore. Das neue Passwort sollte mindestens zwölf zufällige Zeichen ohne erkennbares Muster enthalten.

### 2.2 Kein Schutz gegen Passwort Raten

Der Lehrerlogin vergleicht das Passwort bei jedem Versuch direkt, ohne Zähler, ohne Verzögerung, ohne Sperrung. Ein vierstelliger Code umfasst zehntausend Kombinationen und ist mit einem einfachen Skript in unter einer Minute durchprobiert. Streamlit begrenzt Anfragen von Haus aus nicht.

Empfehlung: Ein Versuchszähler pro Session und Adresse, nach fünf Fehlversuchen eine mehrminütige Sperre. Robuster wäre eine echte Abschaltung vor der App, etwa ein Reverse Proxy mit Basic Auth auf die Lehrerroute. Secrets plus Ratenbegrenzung sind für diesen Zweck jedoch ausreichend.

## 3. Hohe Risiken

### 3.1 Manipulierbare Join URL durch den Host Header

```python
host_url = st.context.headers.get("host", "localhost:8501")
STUDENT_JOIN_URL = f"https://{host_url}"
```

Der Host Header wird vom Client gesendet und ist damit vom Angreifer bestimmt. Ruft jemand die Seite mit gefälschtem Host Header auf, erzeugt die App einen QR Code, der auf die Domain des Angreifers zeigt. Wird dieser Code auf dem Beamer angezeigt, landen die Schüler auf einer nachgebauten Seite. Das Szenario ist als Web Cache Poisoning beziehungsweise Host Header Injection bekannt.

Empfehlung: Die öffentliche URL fest hinterlegen, etwa als Streamlit Secret, statt sie aus dem Request abzuleiten. Alternativ eine Whitelist erlaubter Hostnamen und bei Abweichung eine Ablehnung.

### 3.2 Manipulierbarer Spielzustand

Der gesamte Spielzustand liegt in einem einzigen Dictionary, das über st.cache_resource geteilt wird. Daraus ergeben sich drei Probleme, die einzeln harmlos und zusammen erheblich sind.

Erstens sind Nicknames nicht geschützt. Wer denselben Namen eingibt wie ein Mitschüler, schreibt Einträge unter dessen Identität in die Tabelle all_guesses, weil nur der Name als Schlüssel dient. Damit lassen sich fremde Ergebnisse verderben oder fremde Punkte übernehmen.

Zweitens ist die Tipp Sperre nur im session_state verankert. Der Button LOCK IN GUESS setzt ein Flag in der Browser Session. Öffnet man einen neuen Inkognito Tab, ist das Flag verschwunden und es lässt sich erneut tippen. Nichts auf dem Server verhindert mehrere Tipps pro Person und Runde.

Drittens fehlt eine Frist. Tipps fließen auch dann noch in die Wertung, wenn die Auflösung auf dem Beamer bereits sichtbar ist, solange die Runde im Zähler aktiv ist. Wer die echte Position abliest und schnell einen frischen Tab öffnet, erhält die vollen Punkte.

Empfehlung: Ein zufälliger Join Token pro Schüler, der mit dem Namen gespeichert und bei jedem Tipp mitgeschickt wird. Auf dem Server prüfen, ob für die Kombination aus Runde und Name bereits ein Eintrag existiert, und zusätzliche Einträge verwerfen. Sobald show_leaderboard wahr ist, keine Tipps mehr für die aktuelle Runde annehmen. Jede dieser Maßnahmen umfasst wenige Zeilen.

### 3.3 Rollenprüfung vermutlich nur am Eingang

Der Zugang zur Lehreransicht läuft über den Abfrageparameter role=teacher und das Passwort. Aus dem sichtbaren Code ging nicht vollständig hervor, ob die Authentifizierung vor jedem Lehrerblock erneut geprüft wird oder nur beim ersten Aufruf. Bei Streamlit ist es ein häufiger Fehler, dass nach einem Rerun oder manipuliertem session_state geschützte Bereiche ohne erneute Prüfung erreichbar sind. Zu prüfen ist daher, ob Aktionen wie Upload, Rundenwechsel oder Zurücksetzen nur daran hängen, dass der Codeblock der Lehreransicht ausgeführt wird, und nicht an einer explizit geprüften Variable wie st.session_state.authenticated.

Empfehlung: Eine einzige Variable is_teacher fest im session_state, gesetzt nur nach erfolgreichem Passwortvergleich, und jeder Lehrerblock beginnt mit einer Prüfung dieser Variable.

## 4. Mittlere Risiken

### 4.1 Dateiupload ohne erkennbare Härtung

Der Upload von Ton und Videodateien in den Ordner clips zeigt im geprüften Code keine Validierung von Dateiname, Größe oder Inhaltstyp. Drei konkrete Gefahren: Ein Dateiname mit Pfadanteilen kann außerhalb des Zielordners landen, wenn der Name ungefiltert verwendet wird. Ohne Größenlimit füllt ein einzelner Upload den freien Speicher von Streamlit Cloud, das Limit liegt bei etwa einem Gigabyte, danach ist die Anwendung für alle Nutzer nicht mehr erreichbar. Ohne Endungsprüfung landen beliebige Dateien auf dem Server.

Empfehlung: Den gespeicherten Dateinamen serverseitig neu erzeugen, etwa mit uuid, den Originalnamen nur als Anzeigetext führen. Ein hartes Limit von zwanzig bis fünfzig Megabyte. Endungen strikt auf mp3, m4a, wav und mp4 beschränken, besser noch auf den Mime Typ achten. Den Upload ausschließlich hinter der Lehrerauthentifizierung anbieten.

### 4.2 Globale Steuerfunktionen für Schüler erreichbar

Auch wenn die Oberfläche Steuerfunktionen nur der Lehrkraft anzeigt, ist das ein reines Anzeigeproblem. Streamlit Anwendungen haben kein serverseitiges Rollenkonzept, jede Session führt denselben Code aus. Laufen Aktionen wie Rundenwechsel oder Spiel Zurücksetzen abhängig vom Anzeigepfad statt von der Authentifizierung, kann ein technisch versierter Schüler diese Aktionen auslösen. Die Prüfung gehört zum selben Vorgang wie Punkt 3.3.

## 5. Niedrige Risiken

Die Datei requirements.txt enthält keine Versionsangaben. Für ein Unterrichtsprojekt akzeptabel, ein Pinning verhindert jedoch böse Überraschungen bei Updates von Streamlit.

Nicknames haben keine Längenbegrenzung. Sehr lange Namen verunstalten das Leaderboard, dreißig Zeichen plus eine Beschränkung auf sinnvolle Zeichen sind eine Einzeiler Maßnahme.

app_local.py erzeugt den QR Code mit der lokalen Adresse über unverschlüsseltes http. Im geschützten Schulnetz vertretbar, jeder im selben Netz sieht jedoch den Datenverkehr und kann am Spiel teilnehmen. Für den Betrieb außerhalb des Klassenzimmers wäre die Variante zu deaktivieren.

Das Spielgedächtnis liegt nur im Arbeitsspeicher, ein Neustart löscht alles. Das ist kein Sicherheitsproblem, erklärt aber das Fehlen serverseitiger Schutzmechanismen und wäre ein Argument, den Zustand bei Bedarf in eine sqlite Datei oder kleine Datenbank zu verlagern.

## 6. Priorisierte Reihenfolge der Behebung

An erster Stelle steht das Passwort. Secrets nutzen, starkes Passwort wählen, den alten Wert als endgültig verbrannt betrachten, wenn möglich die Historie bereinigen.

Danach folgt die Härtung der Authentifizierung, also Ratenbegrenzung beim Login und eine durchgängig geprüfte is_teacher Variable vor jedem Lehrerblock, eingeschlossen Upload und Zurücksetzen.

Als nächstes wird die Join URL fest hinterlegt statt aus dem Host Header gebaut.

Daran anschließend die Integrität des Spiels: Join Token, Duplikatprüfung pro Runde und Name, Tipp Sperre ab dem Moment der Auflösung.

Danach die Härtung des Uploads mit neu erzeugten Dateinamen, Größenlimit und Endungsprüfung.

Zum Schluss die kleineren Punkte, also Versionspinning, Namensbegrenzung und das Bewusstsein für den Klartextbetrieb im lokalen Netz.

## 7. Beobachtungen zur Auslieferungsebene

Neben dem Quellcode wurde die live betriebene Instanz besucht, insbesondere die Antworten, die der Server an den Browser schickt, und die dabei gesetzten Cookies in beiden Bereichen der Anwendung, also im Schülerpanel und im Lehrerpanel. Diese Ebene liegt bei Streamlit Community Cloud außerhalb der Kontrolle der Anwendung, viele der folgenden Punkte sind deshalb als dokumentiert und plattformseitig zu verstehen.

### 7.1 Content Security Policy, nicht vorhanden

Es wurde keine Content Security Policy beobachtet. Eine CSP ist die wichtigste Verteidigungslinie gegen die Einschleusung fremder Skripte, also gegen Cross Site Scripting. Bei einer Streamlit Anwendung ist das besonders relevant, weil die Oberfläche aus dynamisch geladenen Komponenten besteht und ein Angreifer, dem eine Einschleusung gelingt, ohne CSP freie Bahn hätte.

Empfehlung: Eine Content Security Policy über den gleichnamigen Header setzen. Ein sinnvoller Start beschränkt default-src auf self und erlaubt explizit nur die Quellen, die die Anwendung tatsächlich braucht, also die Kartenkacheln von OpenStreetMap und die Streamlit eigenen Verbindungen. Der Start erfolgt am besten im Report Only Modus, die Konsole wird auf Verstöße beobachtet und die Richtlinie anschließend geschärft.

### 7.2 X-Frame-Options und Clickjacking, nicht vorhanden

Die Seite lässt sich in einem fremden Frame einbetten, ein Framerahmenschutz fehlt also. Das ermöglicht Clickjacking, also das Überlagern der echten Seite mit einer täuschenden Oberfläche, durch die ein Nutzer unwissentlich Aktionen auslöst. Im Schulkontext ist das Risiko gering, die Behebung kostet jedoch fast nichts.

Empfehlung: Entweder den klassischen Header X-Frame-Options mit dem Wert DENY oder SAMEORIGIN setzen, oder moderner die Angabe frame-ancestors none innerhalb der Content Security Policy aus Punkt 7.1.

### 7.3 Umleitung und fehlende Strict Transport Security

Die erste Umleitung von http zu https geht an einen anderen Host. Dadurch wird der Strict Transport Security Header beim ersten Kontakt verworfen, weil HSTS nur über eine verschlüsselte Verbindung desselben Hosts gültig ist. Zusätzlich wurde beobachtet, dass die Antworten der Instanz überhaupt keinen Strict Transport Security Header enthalten.

Empfehlung: Die Kette so umbauen, dass der erste Sprung auf derselben Domain auf https passiert und erst danach weitere Umleitungen folgen. Bei Streamlit Community Cloud ist darauf wenig Einfluss möglich, der Weg führt über eine eigene Domain mit einem Proxy davor.

### 7.4 Subresource Integrity, nicht vorhanden

Externe Skripte werden verschlüsselt geladen, aber ohne Integritätsprüfung. SRI bedeutet, dass im HTML Attribut ein Hash des erwarteten Skripts steht und der Browser die Datei verwirft, wenn sie verändert wurde. Die praktische Bedeutung ist hier gering, weil alles zumindest verschlüsselt übertragen wird.

Empfehlung: Bei allen externen Script und Link Tags die Attribute integrity und crossorigin ergänzen. Da eine Streamlit Anwendung diese Tags nicht selbst schreibt, hilft hier vor allem die CSP aus Punkt 7.1.

### 7.5 Referrer Policy, nicht gesetzt

Der Header fehlt in den beobachteten Antworten. Beim Klicken auf externe Links kann dadurch die vollständige Zieladresse inklusive eventueller Parameter an die fremde Seite übermittelt werden.

Empfehlung: Den Header Referrer-Policy auf strict-origin-when-cross-origin setzen.

### 7.6 Cross Origin policies, nicht gesetzt

Die drei Header Cross Origin Embedder Policy, Cross Origin Opener Policy und Cross Origin Resource Policy fehlen. Sie verhindern unter anderem, dass fremde Seiten Fensterbeziehungen zur Anwendung aufbauen oder ihre Ressourcen einbetten.

Empfehlung: Für die Opener Policy der Wert same-origin, für die Embedder Policy require-corp oder credentialless, für die Resource Policy same-origin. Die Einstellungen erst im Beobachtungsmodus testen, weil die Folium Karten externe Kacheln laden.

### 7.7 Cookies, gemischtes Bild

Die Streamlit eigenen Session Cookies sind mit Secure, HttpOnly und SameSite korrekt abgesichert. Auffällig ist dagegen ein Cookie mit dem Namen proxy-tracking-id, das ohne HttpOnly und ohne Secure Flag gesetzt wird. Es stammt aus der Infrastruktur der Plattform und ist von der Anwendung selbst nicht steuerbar. Da dieses Cookie eher der Verbindungsverfolgung dient als eine echte Session zu tragen, ist der Schaden im konkreten Fall überschaubar. Unangenehm ist die Kombination mit der fehlenden Content Security Policy aus Punkt 7.1. Der Abhilfe Weg ist derselbe wie dort, also eine eigene Domain mit einem Proxy davor.

### 7.8 Weitere positive Beobachtungen

Es gibt keine offenen CORS Freigaben und die Hauptseite enthält den nosniff Header. Auffällig war, dass dieser Header auf den statischen Asset Routen unter build und assets fehlt. Das Risiko ist gering. Bei einem späteren Wechsel hinter einen eigenen Proxy gehört der Header auf jede Antwort.

### 7.9 Serverinformationen

Beim Betrachten der Verbindungsdaten wurde sichtbar, dass hinter der Anwendung Google Cloud, ein Content Delivery Network und ein Nginx stehen, wobei die Versionsnummer des Nginx erkennbar ist. Da die Serverinfrastruktur bei Streamlit Community Cloud nicht kontrollierbar ist, verbleibt der Punkt als dokumentierter Hinweis.

### 7.10 Was bei der Prüfung nicht auffiel

Es fielen keine offenen Debug Endpunkte auf, keine Verzeichnislisten, keine sichtbaren Fehlermeldungen mit internen Details, keine unverschlüsselt übertragenen Passwörter und keine privaten Schlüssel in den Antworten. Die relevanten Angriffsflächen dieser Anwendung sind ohnehin nicht die klassischen Web Schwachstellen, weil es keine Datenbank gibt und keine serverseitige Abfrage, die Benutzereingaben in SQL oder Shell Befehle übersetzt. Eine tiefgehende Prüfung mit Einbruchstechniken wird erst dann sinnvoll, wenn eine Datenbank oder echte Authentifizierung mit Token nachgerüstet wird.

## 8. Gesamtbewertung

Das Bild ist zweigeteilt. Innerhalb des Anwendungscodes liegen die kritischen Themen bei Passwort, Ratenbegrenzung und Zustandsmanipulation. Auf der Auslieferungsebene fehlen vor allem Content Security Policy und Framerahmenschutz, und ein Tracking Cookie der Plattform ist unzureichend abgesichert. Nach dem Passwort und der Authentifizierung folgt als dritte Maßnahme die Content Security Policy zusammen mit frame-ancestors, danach die fest hinterlegte Join URL, die Spielintegrität und der Upload. Die Codebasis ist für ein mit KI Unterstützung entstandenes Wochenendprojekt solide strukturiert. Die vorübergehende Abschaltung der Instanz ist der richtige Schritt, um die Korrekturen in Ruhe umzusetzen, bevor die Anwendung wieder öffentlich erreichbar ist.
