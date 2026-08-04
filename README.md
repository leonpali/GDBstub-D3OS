<div align="center">
  <img src="https://coconucos.cs.hhu.de/lehre/bigdata/resources/img/hhu-logo.svg" width=300>

  [![Download](https://img.shields.io/static/v1?label=&message=pdf&color=EE3F24&style=for-the-badge&logo=adobe-acrobat-reader&logoColor=FFFFFF)](/../-/jobs/artifacts/master/file/document/thesis.pdf?job=latex)
</div>

# :notebook: &nbsp; Aufgabenbeschreibung

Ziel dieser Bachelorarbeit ist die Implementierung eines GDB-Stubs für D3OS. Dieser soll es ermöglichen, das Betriebssystem und darauf ausgeführten Code auf echter Hardware zu debuggen, indem ein Host-OS, auf dem der GNU Debugger läuft, über das GDB Remote Serial Protocol mit D3OS kommuniziert.<br> 
Als Grundlage für die Kommunikation wird die gdbstub Crate in D3OS integriert. Der GDB-Stub wird dabei in den Kernel von D3OS integriert, um direkten Zugriff auf Prozessorzustand, Speicher, sowie Exceptions und Interrupts zu erhalten. Die Kommunikation erfolgt über eine serielle Schnittstelle, deren bestehende Implementierung im Rahmen dieser Arbeit analysiert und verbessert werden muss, da eingehende Daten derzeit nicht zuverlässig verarbeitet werden.<br>
Die Implementierung soll grundlegende Debugging Funktionalitäten unterstützen, wie das Setzen von Breakpoints, Auslesen von Registern und Speicher und die zeilenweise Ausführung von Code durch steps.<br>
Abschließend wird die entwickelte Lösung anhand von Testszenarien evaluiert. Dabei wird überprüft, inwiefern D3OS sich zuverlässig aus einer GDB Sitzung heraus steuern und analysieren lässt. Einschränkungen sowie mögliche Erweiterungen werden diskutiert.

# :notebook: &nbsp; Kompilieren
Verwendete rustc Version: rustc 1.94.0-nightly (8d670b93d 2025-12-31)
Verwendete Cargo Version: cargo 1.94.0-nightly (b54051b15 2025-12-30)
Verwendete Toolchain: nightly-2026-01-01-x86_64-unknown-linux-gnu

Abgesehen von den Crates gdbstub und gdbstub_arch, die bereits in der Cargo.toml gelistet sind wurden keine weiteren Bibliotheken zu D3OS hinzugefügt.
Das Projekt kann durch den Befehl "cargo make --no-workspace gdbstub" kompiliert und direkt in QEMU gestartet werden. 

# :notebook: &nbsp; Nutzung
Zur Nutzung auf echter Hardware muss die Version des Projekts in der Branch "abgabe" verwendet werden. Wenn sie kompiliert wurde, kann das Image durch zum Beispiel balenaEtcher auf einen USB-Stick geflashed werden. Der Stub ist standardmäßig auf Com3 eingestellt, also muss im BIOS des Rechners ebenfalls Com3 aktiv sein. GDB kann entweder über ein Terminal mit "target remote [GERÄTENAME]" oder über die VS Code Konfiguration "gdbstub" verbunden werden. In der VS Code launch.json muss hier auch das entsprechende Gerät angegeben werden. Die Rechner müssen über einen seriellen Anschluss verbunden sein. Weitere Befehle sind im Kapitel Nutzung der Ausarbeitung zu finden.

In der Branch "development" ist der Stub mit der aktuellen Version von D3OS. Diese funktioniert nur in QEMU zuverlässig und nicht auf Hardware. GDB kann hier über den Befehl "target remote localhost:4321" verbunden wereden. Das geht sowohl im Terminal, als auch über die VS Code launch.json. Weitere Befehle sind im Kapitel Nutzung der Ausarbeitung zu finden.

Um die Funktionen schnell zu testen gibt es in der Datei os/kernel/src/serialtest/testserial.rs die Funktion gdb_break_here(), die Variablen und Structs instanziiert und verschiedene Funktionen aufruft. Dadurch können die implementierten Funktionen einfach getestet werden.