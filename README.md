<div align="center">
  <img src="https://coconucos.cs.hhu.de/lehre/bigdata/resources/img/hhu-logo.svg" width=300>

  [![Download](https://img.shields.io/static/v1?label=&message=pdf&color=EE3F24&style=for-the-badge&logo=adobe-acrobat-reader&logoColor=FFFFFF)](/../-/jobs/artifacts/master/file/document/thesis.pdf?job=latex)
</div>

# :notebook: &nbsp; Aufgabenbeschreibung

Ziel dieser Bachelorarbeit ist die Implementierung eines GDB-Stubs für D3OS. Dieser soll es ermöglichen, das Betriebssystem und darauf ausgeführten Code auf echter Hardware zu debuggen, indem ein Host-OS, auf dem der GNU Debugger läuft, über das GDB Remote Serial Protocol mit D3OS kommuniziert.<br> 
Als Grundlage für die Kommunikation wird die gdbstub Crate in D3OS integriert. Der GDB-Stub wird dabei in den Kernel von D3OS integriert, um direkten Zugriff auf Prozessorzustand, Speicher, sowie Exceptions und Interrupts zu erhalten. Die Kommunikation erfolgt über eine serielle Schnittstelle, deren bestehende Implementierung im Rahmen dieser Arbeit analysiert und verbessert werden muss, da eingehende Daten derzeit nicht zuverlässig verarbeitet werden.<br>
Die Implementierung soll grundlegende Debugging Funktionalitäten unterstützen, wie das Setzen von Breakpoints, Auslesen von Registern und Speicher und die zeilenweise Ausführung von Code durch steps.<br>
Abschließend wird die entwickelte Lösung anhand von Testszenarien evaluiert. Dabei wird überprüft, inwiefern D3OS sich zuverlässig aus einer GDB Sitzung heraus steuern und analysieren lässt. Einschränkungen sowie mögliche Erweiterungen werden diskutiert.

