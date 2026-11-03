# RevAger – Umstellung auf Java 21

RevAger ist eine Java-Swing-Anwendung zur Unterstützung von Software-Reviews (Version 1.4.3).
Ursprünglich wurde das Projekt für **Java 8** entwickelt. Dieses Dokument beschreibt, was nötig war,
damit es mit **Java 21** kompiliert und läuft, und wie man die Schritte auf einem anderen Rechner
wiederholt.

---

## 1. Ausgangslage: Was war kaputt?

Nach dem Öffnen in VS Code mit installiertem JDK 21 zeigte das Problems-Panel **259 Fehler**, alle
in denselben Dateien:

- `src/org/revager/app/model/schema/*` (generierte XML-Klassen)
- `src/org/revager/io/impl/XMLResiIO.java`
- `src/org/revager/io/impl/XMLResiValidationEventHandler.java`

Typische Meldungen:

```
The import javax.xml.bind cannot be resolved
XmlElement cannot be resolved to a type
JAXBContext cannot be resolved
The method unmarshal(String) of type Adapter1 must override or implement a supertype method
```

### Ursache

Das Projekt verwendet **JAXB** (`javax.xml.bind`), um Review-Dateien als XML zu lesen und zu
schreiben. JAXB war bis Java 8 **Teil des JDK**. Mit **Java 11** wurde es entfernt
([JEP 320](https://openjdk.org/jeps/320)). Auf Java 21 fehlen die Klassen deshalb, und alles, was
sie benutzt, lässt sich nicht kompilieren. Die letzte Meldung oben ist nur ein Folgefehler: Die
Oberklasse `XmlAdapter` wird nicht gefunden.

---

## 2. Was geändert wurde

| Datei | Änderung |
|---|---|
| `lib/` | 5 neue JAR-Dateien (JAXB und Abhängigkeiten, siehe Abschnitt 3) |
| `.classpath` | JRE von `JavaSE-1.8` auf `JavaSE-21` umgestellt, die 5 JARs eingetragen |
| `build.xml` | `source`/`target` von `1.8` auf `21`, JARs im `RevAger.classpath` ergänzt, falschen SwingX-Pfad korrigiert (`lib-dev` → `lib`) |
| `.vscode/settings.json` | `java.configuration.runtimes`: legt fest, welches installierte JDK mit `JavaSE-21` gemeint ist (siehe Abschnitt 5) |

**Am Java-Quellcode wurde nichts geändert.**

---

## 3. Wie findet man heraus, welche JARs man braucht?

Das ist der wichtigste Teil. Die Vorgehensweise funktioniert genauso für andere fehlende
Bibliotheken.

### Schritt 3.1: Den fehlenden Paketnamen aus der Fehlermeldung lesen

```
The import javax.xml.bind cannot be resolved
```

Das fehlende Paket ist `javax.xml.bind`. Fehlermeldungen vom Typ "cannot be resolved" sagen also
genau, *was* fehlt, aber nicht, *in welcher JAR-Datei* es steckt.

### Schritt 3.2: Herausfinden, warum das Paket fehlt

Für Pakete, die mit `javax.` beginnen, gilt: Wenn sie unter Java 8 funktioniert haben und unter
Java 11+ nicht mehr, wurden sie mit großer Wahrscheinlichkeit durch JEP 320 aus dem JDK entfernt.
Die wichtigsten davon:

| Paket | Entferntes JDK-Modul | Ersatz (Maven-Artefakt) |
|---|---|---|
| `javax.xml.bind` | `java.xml.bind` (JAXB) | `jakarta.xml.bind:jakarta.xml.bind-api` + `org.glassfish.jaxb:jaxb-runtime` |
| `javax.activation` | `java.activation` | `com.sun.activation:jakarta.activation` |
| `javax.xml.ws` | `java.xml.ws` (JAX-WS) | `jakarta.xml.ws:jakarta.xml.ws-api` + `com.sun.xml.ws:jaxws-rt` |
| `javax.annotation` | `java.xml.ws.annotation` | `jakarta.annotation:jakarta.annotation-api` |

### Schritt 3.3: API und Implementierung unterscheiden

Viele Java-Standards bestehen aus **zwei Teilen**:

- **API**: nur die Schnittstellen und Annotationen (`JAXBContext`, `@XmlElement` …). Damit
  **kompiliert** der Code.
- **Implementierung**: der Code, der die Arbeit tatsächlich erledigt. Ohne ihn **startet** das
  Programm zwar, stürzt aber beim ersten XML-Zugriff ab, typischerweise mit:
  ```
  javax.xml.bind.JAXBException: Implementation of JAXB-API has not been found on module path or classpath.
  ```

**Merke:** Nur die API einzubinden reicht nicht, auch wenn die IDE danach keine Fehler mehr zeigt.

### Schritt 3.4: Die richtige Version wählen (Paketname beachten)

JAXB gibt es in zwei Generationen mit **unterschiedlichen Paketnamen**:

| Version | Paketname im Code | Passt zu RevAger? |
|---|---|---|
| 2.3.x | `javax.xml.bind` | ✅ ja, keine Code-Änderung nötig |
| 3.x / 4.x | `jakarta.xml.bind` | ❌ nur nach Umschreiben aller Imports |

Weil der RevAger-Code `import javax.xml.bind…` verwendet, wurde **2.3.x** gewählt. Dabei ist
verwirrend: Das Artefakt heißt `jakarta.xml.bind-api`, enthält in Version 2.3.3 aber noch das
Paket `javax.xml.bind`.

### Schritt 3.5: Die Abhängigkeiten der Implementierung ermitteln

Eine JAR-Datei braucht oft weitere JARs. Welche das sind, steht in ihrer **POM-Datei** auf
[Maven Central](https://repo1.maven.org/maven2/), dem zentralen Verzeichnis für
Java-Bibliotheken. Die POM liegt immer neben der JAR:

```
https://repo1.maven.org/maven2/org/glassfish/jaxb/jaxb-runtime/2.3.9/jaxb-runtime-2.3.9.pom
```

Dort im Abschnitt `<dependencies>` nachsehen und dabei so filtern:

| Eintrag in der POM | Benötigt? |
|---|---|
| ohne `<scope>` oder `<scope>compile</scope>` | ✅ ja |
| `<scope>runtime</scope>` | ✅ ja (wird zur Laufzeit gebraucht) |
| `<scope>provided</scope>` | ❌ nein (soll "von außen" kommen, siehe unten) |
| `<scope>test</scope>` | ❌ nein |
| `<optional>true</optional>` | ❌ nein (nur für Sonderfunktionen) |

Für `jaxb-runtime-2.3.9` ergibt das:

| Abhängigkeit | Scope | Ergebnis |
|---|---|---|
| `jakarta.xml.bind-api` | compile | ✅ |
| `txw2` | compile | ✅ |
| `istack-commons-runtime` | compile | ✅ |
| `com.sun.activation:jakarta.activation` | runtime | ✅ |
| `jakarta.activation-api` | provided | ❌ (in `jakarta.activation` bereits enthalten) |
| `stax-ex`, `FastInfoset` | optional | ❌ |
| `junit` | test | ❌ |

**Fehlende Versionsnummern:** Steht in der POM bei einer Abhängigkeit keine `<version>`, ist sie
in einer übergeordneten POM festgelegt (`<parent>` oder eine "BOM"). Für JAXB 2.3.9 ist das
[`jaxb-bom-2.3.9.pom`](https://repo1.maven.org/maven2/org/glassfish/jaxb/jaxb-bom/2.3.9/jaxb-bom-2.3.9.pom).
Dort stehen zum Beispiel `istack.version = 3.0.12` und `jakarta.activation.version = 1.2.2`.

Dieselbe Prüfung macht man anschließend für jede neu hinzugekommene JAR, also "rekursiv", bis
keine neuen Abhängigkeiten mehr auftauchen.

> **Abkürzung mit Maven:** Ist [Maven](https://maven.apache.org/) installiert, kann Maven diese
> Suche automatisch erledigen. Dafür eine Datei `pom.xml` in einem leeren Ordner anlegen, in der
> nur `jaxb-runtime` als Abhängigkeit steht, und dort ausführen:
> `mvn dependency:copy-dependencies -DoutputDirectory=jars`.
> Im Ordner `jars` liegen dann alle benötigten Dateien. Zusätzlich liefert Maven noch
> `jakarta.activation-api-1.2.2.jar`. Die ist doppelt, aber harmlos, weil `jakarta.activation`
> dieselben Klassen schon enthält. Die Prüfung von Hand oben zeigt, was dabei im Hintergrund
> passiert.

### Schritt 3.6: Prüfen, ob eine Klasse wirklich in einer JAR steckt

```bash
unzip -l lib/jakarta.xml.bind-api-2.3.3.jar | grep JAXBContext
# → javax/xml/bind/JAXBContext.class  ✅
```

### Ergebnis: die 5 benötigten JARs

| Datei | Zweck | Download |
|---|---|---|
| `jakarta.xml.bind-api-2.3.3.jar` | JAXB-API (`javax.xml.bind.*`) | [Link](https://repo1.maven.org/maven2/jakarta/xml/bind/jakarta.xml.bind-api/2.3.3/jakarta.xml.bind-api-2.3.3.jar) |
| `jaxb-runtime-2.3.9.jar` | JAXB-Implementierung | [Link](https://repo1.maven.org/maven2/org/glassfish/jaxb/jaxb-runtime/2.3.9/jaxb-runtime-2.3.9.jar) |
| `txw2-2.3.9.jar` | XML-Schreiben (von jaxb-runtime benötigt) | [Link](https://repo1.maven.org/maven2/org/glassfish/jaxb/txw2/2.3.9/txw2-2.3.9.jar) |
| `istack-commons-runtime-3.0.12.jar` | Hilfsbibliothek von jaxb-runtime | [Link](https://repo1.maven.org/maven2/com/sun/istack/istack-commons-runtime/3.0.12/istack-commons-runtime-3.0.12.jar) |
| `jakarta.activation-1.2.2.jar` | `javax.activation` (ebenfalls aus dem JDK entfernt) | [Link](https://repo1.maven.org/maven2/com/sun/activation/jakarta.activation/1.2.2/jakarta.activation-1.2.2.jar) |

Lizenz aller fünf: [Eclipse Distribution License 1.0](https://www.eclipse.org/org/documents/edl-v10.php).

---

## 4. Schritt für Schritt nachmachen

### Voraussetzungen

- JDK 21 installiert, zum Beispiel [Eclipse Temurin 21](https://adoptium.net/).
  Prüfen mit `java -version`.
- VS Code mit dem "Extension Pack for Java".

### 4.1 JARs herunterladen

Entweder die Links aus der Tabelle oben im Browser öffnen und die Dateien nach `lib/` verschieben,
oder im Terminal:

```bash
cd lib
B=https://repo1.maven.org/maven2
curl -LO $B/jakarta/xml/bind/jakarta.xml.bind-api/2.3.3/jakarta.xml.bind-api-2.3.3.jar
curl -LO $B/org/glassfish/jaxb/jaxb-runtime/2.3.9/jaxb-runtime-2.3.9.jar
curl -LO $B/org/glassfish/jaxb/txw2/2.3.9/txw2-2.3.9.jar
curl -LO $B/com/sun/istack/istack-commons-runtime/3.0.12/istack-commons-runtime-3.0.12.jar
curl -LO $B/com/sun/activation/jakarta.activation/1.2.2/jakarta.activation-1.2.2.jar
cd ..
```

### 4.2 `.classpath` anpassen (für VS Code / Eclipse)

JRE-Eintrag ändern:

```xml
<classpathentry kind="con" path="org.eclipse.jdt.launching.JRE_CONTAINER/org.eclipse.jdt.internal.debug.ui.launcher.StandardVMType/JavaSE-21"/>
```

Unter dem `swingx`-Eintrag ergänzen:

```xml
<classpathentry kind="lib" path="lib/jakarta.xml.bind-api-2.3.3.jar"/>
<classpathentry kind="lib" path="lib/jaxb-runtime-2.3.9.jar"/>
<classpathentry kind="lib" path="lib/txw2-2.3.9.jar"/>
<classpathentry kind="lib" path="lib/istack-commons-runtime-3.0.12.jar"/>
<classpathentry kind="lib" path="lib/jakarta.activation-1.2.2.jar"/>
```

### 4.3 `build.xml` anpassen (für Ant)

```xml
<property name="target" value="21" />
<property name="source" value="21" />
```

```xml
<!-- vorher: ${dir.lib.dev}/swingx-1.6.1.jar – die Datei liegt aber in lib/ -->
<property name="lib.swingx" location="${dir.lib}/swingx-1.6.1.jar" />
```

Im Block `<path id="RevAger.classpath">` vor `</path>`:

```xml
<fileset dir="${dir.lib}" includes="jakarta.*.jar,jaxb-runtime-*.jar,txw2-*.jar,istack-commons-runtime-*.jar" />
```

In die fertige Programm-JAR muss man nichts zusätzlich eintragen: Das Ant-Ziel `unzip-libs` packt
automatisch alle JARs aus `lib/` hinein.

### 4.4 VS Code das richtige JDK zuweisen

Siehe Abschnitt 5. Ohne diesen Schritt startet die App nicht.

### 4.5 VS Code neu einlesen lassen

**Cmd+Shift+P** (Windows/Linux: **Ctrl+Shift+P**) → `Java: Clean Java Language Server Workspace`
→ **Restart and delete**. Danach warten, bis unten "Java: Ready" steht.

### 4.6 Testen

1. Im Problems-Panel (**Cmd+Shift+M**) dürfen keine roten Fehler mehr stehen. Die gelben
   Warnungen sind harmlos.
2. `src/org/revager/Main.java` öffnen und über der `main`-Methode auf **Run** klicken.
3. In der App ein Review anlegen, **speichern**, schließen und **wieder öffnen**. Genau dabei wird
   JAXB benutzt.

---

## 5. Stolperfalle: `UnsupportedClassVersionError`

Nach den Schritten oben stürzte die App beim Start zunächst so ab:

```
java.lang.UnsupportedClassVersionError: org/revager/Main has been compiled by a more recent
version of the Java Runtime (class file version 69.0), this version of the Java Runtime only
recognizes class file versions up to 65.0
```

**Ursache:** Die Java-Erweiterung von VS Code bringt ein **eigenes JDK** mit (bei Version 1.58:
Java 25). In `.classpath` steht `JavaSE-21`, aber VS Code wusste nicht, welches installierte JDK
damit gemeint ist. Deshalb hat es mit dem mitgebrachten Java 25 **kompiliert**, und **gestartet**
wurde die App mit dem installierten Java 21. Java 21 kann Klassen von Java 25 nicht lesen.

| Class-File-Version | Java-Version |
|---|---|
| 52 | 8 |
| 55 | 11 |
| 61 | 17 |
| 65 | 21 |
| 69 | 25 |

**Lösung:** In `.vscode/settings.json` festlegen, welches JDK zu `JavaSE-21` gehört:

```json
{
  "explorer.compactFolders": false,
  "java.configuration.runtimes": [
    {
      "name": "JavaSE-21",
      "path": "/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home",
      "default": true
    }
  ]
}
```

> ⚠️ Der Pfad ist **rechnerabhängig**. Auf macOS findet man ihn mit `/usr/libexec/java_home -v 21`.
> Unter Windows ist es zum Beispiel `C:\\Program Files\\Eclipse Adoptium\\jdk-21…`.

Danach noch einmal `Java: Clean Java Language Server Workspace` ausführen und mit diesem Befehl
prüfen:

```bash
javap -v bin/org/revager/Main.class | grep major
# erwartet: major version: 65
```

---

## 6. Git: neue JARs werden ignoriert

In `.gitignore` steht `*.jar`. Neue JAR-Dateien werden deshalb von Git **nicht** erfasst. Die
bestehenden JARs sind nur im Repository, weil sie früher erzwungen hinzugefügt wurden. Für die
neuen JARs muss man das genauso machen:

```bash
git add -f lib/jakarta.xml.bind-api-2.3.3.jar lib/jaxb-runtime-2.3.9.jar lib/txw2-2.3.9.jar \
           lib/istack-commons-runtime-3.0.12.jar lib/jakarta.activation-1.2.2.jar
git add .classpath build.xml .vscode/settings.json README.md
```

Ohne `-f` fehlen die JARs bei allen, die das Projekt klonen.

---

## 7. Bekannte offene Punkte

- **Gamecontroller auf Apple Silicon:** `lib/jinput-native-files/libjinput-osx.jnilib` enthält
  nur Code für ppc, i386 und x86_64, nicht für arm64. Auf M-Macs funktioniert die
  Gamecontroller-Funktion mit einem arm64-JDK deshalb voraussichtlich nicht.
- **Warnungen:** Rund 150 Warnungen (raw `JComboBox`, veraltete Klassen wie `Observable`/`Observer`,
  `getMenuShortcutKeyMask()` …) bleiben bestehen. Sie verhindern weder das Kompilieren noch das
  Starten.
- **Ant** muss für einen Build außerhalb der IDE separat installiert werden
  (macOS: `brew install ant`). Danach `ant dist` ausführen, das Ergebnis liegt in `dist/`.
- `.project` wird von der VS-Code-Java-Erweiterung automatisch um einen `filteredResources`-Block
  ergänzt. Das ist harmlos und muss nicht committet werden.
