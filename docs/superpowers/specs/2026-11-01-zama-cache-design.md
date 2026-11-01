# Zama-Cache für mehrere Wallets

## Ziel und Umfang

Ein Yarn-Monorepo enthält ein TypeScript-SDK, eine API und einen privaten Datenbankdienst. Beide Dienste erhalten eigene Docker-Images. Anwendungen können ihren bisherigen Zama-Decrypt direkt aufrufen oder ihn dem SDK als verzögerten Callback übergeben. Der PoC soll wiederholte fhEVM-Entschlüsselungen vermeiden und die Ergebnisse je Login-Wallet isolieren.

## SDK

Local Storage und Datenquelle sind unabhängig konfigurierbar. Die Quelle ist entweder der Decrypt-Callback der Anwendung oder die API. Der lokale Cache ist aus, sessionStorage oder localStorage. Ohne lokalen Cache und mit Callback führt jeder Abruf direkt zum Callback. Im Servermodus gibt es keinen automatischen lokalen Decrypt bei einem API-Fehler. Lokale Treffer dürfen offline gelesen werden, sobald die Wallet lokal entsperrt ist.

decrypt und decryptMany arbeiten mit Chain, Contract und Handle. Der Browser trennt Einträge zusätzlich nach Wallet und Datenquelle. Der Batch liefert je Eingabeposition einen typisierten Erfolg oder Fehler. Er dedupliziert intern, prüft Treffer und bündelt nur die zulässigen Misses pro kompatiblem Entschlüsselungskontext. Ein fehlerhafter Handle darf andere Ergebnisse nicht vernichten. Bei unteilbarem Batchfehler fehlerhafte Teilmengen durch begrenztes Aufteilen isolieren, sofern es sich um einen handlebezogenen Fehler handelt. Infrastrukturfehler als wiederholbar kennzeichnen. Werte einschließlich großer Ganzzahlen verlustfrei serialisieren.

## Lokaler Schlüsselschutz

Gewählt ist Wallet plus Gerätefreigabe. Der Server als Schlüsselquelle hätte den Login vereinfacht, aber den lokalen Modus beim Neustart von der API abhängig gemacht. Ein von der Anwendung gelieferter Schlüssel würde die Schlüsselverwaltung jedem Integrator überlassen.

Für localStorage bindet das SDK beim Einrichten eine lokal verifizierte Wallet-Challenge an eine Passkey-Credential derselben Origin. Ein zufälliger Datenschlüssel pro Wallet wird mit einem aus der WebAuthn-PRF abgeleiteten Schlüssel verpackt. Nur Salt, Credential-ID, Nonces, verpackter Schlüssel und verschlüsselte Einträge werden persistent gespeichert. Zum erneuten Entsperren Wallet-Besitz und Passkey mit Nutzerverifikation prüfen. Die Wallet-Signatur ist kein Schlüsselmaterial. Fehlende PRF-Unterstützung explizit melden. Bei verlorenem Credential kann der Nutzer den Cache verwerfen und neu aufbauen. Für sessionStorage liegt der Schlüssel ausschließlich im Speicher des Tabs. Ein Reload beginnt daher mit leerem nutzbarem Session-Cache. Bei Logout oder Wallet-Wechsel Schlüssel und Klartextreferenzen verwerfen und laufende Antworten an ihren ursprünglichen Kontext binden.

## Autorisierung und Kosten

Die API erlaubt konfigurierte Wallet-Adressen aus der Umgebung. Login mit einmaliger, kurzlebiger Challenge, gebunden an Domain und Wallet, und konfigurierbarer Sessiondauer. Die API bestimmt die Identität aus der Session. Einen privaten Signierschlüssel besitzt ausschließlich die Server-Seite. Bei erstmaliger Freigabe muss die API die Handle-Berechtigung der Login-Wallet, den Contract-Kontext und bei abweichendem Signer die gültige Delegation an die konfigurierte Server-Wallet aus vertrauenswürdiger Quelle prüfen. Eine eigenständige Berechtigung des Servers ersetzt diesen Nachweis nicht. Ist ein Nachweis nicht verlässlich verfügbar, Zugriff verweigern. Vor Implementierung die konkreten Prüfmethoden des gewählten Zama-Netzes verifizieren.

Gewählt ist ein verschlüsselter Wert pro Chain, Contract und Handle mit separaten Freigaben pro Wallet und Wertgeneration. Separate Werte pro Wallet wären einfacher, würden aber Daten und Decrypt-Kosten duplizieren. Ein vorhandener Wert erlaubt nach erfolgreich geprüfter Erstberechtigung eine zusätzliche Freigabe ohne erneuten Decrypt. Eine bestehende Freigabe erlaubt derselben authentifizierten Wallet auch nach Widerruf den Zugriff auf genau diese gespeicherte Generation. Fremde Wallets erhalten bei fehlender Berechtigung denselben Fehler für Treffer und Miss.

Pro Schlüssel parallele Misses zusammenführen. Jede Anfrage bleibt separat autorisiert. Wert und erste Freigabe atomar anlegen. Neue Freigaben atomar an die noch gültige Generation binden. Nur erfolgreiche Lesezugriffe aktualisieren lastUsed. Die konfigurierbare Inaktivitätsfrist ist unabhängig von der Login-Dauer. Null bedeutet unbegrenzt. Ablauf beim Lesen durchsetzen, auch wenn die physische Bereinigung erst später läuft. Mit einer Generation erlöschen deren Freigaben. Lokale Kopien führen ein eigenes lastUsed, Offline-Lesen verlängert die Serverfrist nicht.

## Dienste und Schutzgrenzen

Die API verschlüsselt Werte vor dem Schreiben authentifiziert mit ihrem eigenen Schlüssel. Der private DB-Dienst verschlüsselt diesen Ciphertext zusätzlich mit einem unabhängigen Schlüssel. Beim Lesen entfernt jeder Dienst nur seine eigene Schicht. Nonces pro Verschlüsselung frisch erzeugen, Datensatzidentität und Generation kryptografisch binden. API und DB erhalten getrennte Secrets, nur der DB-Dienst erhält sein Datenvolume. Im PoC können die Secrets aus getrennten, nicht eingecheckten Env-Dateien kommen. Keine Klartexte, Signaturen oder Schlüssel protokollieren. Authentifizierte, transportverschlüsselte Verbindungen außerhalb einer rein lokalen Entwicklungsumgebung.

Dies schützt gestohlene Datenbestände. Eine kompromittierte laufende API kann ihren DB-Zugang und ihre Schlüssel benutzen. Aktives Schadskript in einer entsperrten Browser-Origin kann Klartext abgreifen. Bereits ausgelieferter Klartext kann nicht zurückgerufen werden. Schutz gegen diese Fälle ist kein Ziel des PoC.

## Entscheidungen für die Umsetzung und Abnahme

Dem Implementierer überlassen sind DB-Technik, API-Framework, konkrete Methoden- und Fehlernamen, begrenzte Batchgröße und Bereinigungsintervall. Für den PoC genügt eine API-Instanz. Zentrale Abnahmeszenarien sind zwei berechtigte Wallets mit einem einzigen Decrypt, verweigerter Fremdzugriff auch bei Treffern, Widerruf vor und nach Cache-Ablauf, Teilfehler und Duplikate im Batch sowie lokales Entsperren nach Neustart und Isolation beim Wallet-Wechsel.
