
![Tool Overview](http://www.brendangregg.com/Perf/linux_observability_tools.png)

## Tools
* `htop`
* `mpstat`, `vmstat`, `iostat` vom Package `sysstat`

## Probleme & Lösungen

### Index
* CPU
  * [Vollständigen Systemüberblick erhalten]
  * [Überblick loggen]
  * [Per-Core Auslastung loggen]
  * [Fehlender Parallelismus]
  * [Hohe %IRQ-Auslastung]
  * [Fine-grained Profiling mit perf]
* Memory
* Disk IO
* Network IO


### CPU

#### Vollständigen Systemüberblick erhalten
**Tool:** `htop`

`htop` gibt einen vollständigen Snapshot des Systemperformancestatus, protokolliert diesen jedoch nicht über einen Zeitraum. Es ist daher gut dafür geeignet einen
umfangreichen Überblick über das System und dessen Prozesse zu erhalten.

Bedeutung der Standardspalten:
* **PID** - Prozess-Id des Prozesses
* **USER** - Unix-Benutzername des Besitzers des Prozesses
* **PRI** (_priority_) - Priorität des Prozesses. Je kleiner desto höher die Priorität
* **NI** (_nice_) - Nice-Wert des Prozesses. Beeinflusst die Priorität
* **VIRT** - Gesamte Menge des benutzen virtuellen Speichers den der Prozess besitzt
* **RES** (_resident_) - Wie viel _physikalischen_ Speicher der Prozess tatsächlich gerade belegt (gemessen in KiB when ohne Suffix)
* **SHR** (_shared_) - Wie viel geteilten Speicher der Prozess benutzt (bswp. Shared Libraries)
* **S** (_status_) - Derzeitiger Status des Prozesses
	
	| Buchstabe | Bedeutung   |
	| ----------| ----------- |
	| `R`       | Running: Wird ausgeführt oder mindestens in der Runqueue |
	| `S`       | Interrupible Sleep: Schläft, wartet auf Event oder Signal |
	| `D`, (`W` vor Linux 2.6.0) | Uninterrutible Sleep / Disk: Schläft aber kann nicht durch Signale geweckt werden. Wartet bswp. auf IO für ein Pagefault. _Prozess kann nicht terminiert werden._ |
	| `Z`       | Zombie: Terminiert aber noch nicht vom Parent "gereaped". Sollte nur kurzzeitig existieren, sonst evtl. Bug im Programm. |
	| `T`       | Stoppped by Signal: Durch ein `STOP` Signal (bswp. Ctrl+Z) pausiert. Kann durch ein `CONT` Signal wieder gestartet werden. |
	| `t` (Linux 2.6.33+) | Stopped by Debugger: Durch einen Debugger oder trace gestoppt. |
	| `X`, `x`  | Dead: _Nomen est omen_. Sollte eigentlich nicht in der Liste vorkommen. |
	| `W` (Linux 2.6.33 bis 3.13) | Waking |
	| `P` (Linux 3.9 bis 3.13) | Parked |

* **CPU%** - Durchschnittliche CPU-Nutzung in Prozent im letzen Sample
* **MEM%** - Prozent des physikalischen Speichers (resident) den der Prozess benutzt
* **TIME+** - Totale verbrauchte User- und Systemzeit des Prozesses
* **COMMAND** - Commandline die den Prozess gestartet hat

#### Überblick loggen
`vmstat` loggt zyklisch einen gröberen Überblick über das System.
Der Datensatz ist nicht per Prozess und auch weniger ausführlich als
der von `htop` jedoch kann er über einen längeren Zeitraum zyklisch
beobachtet und abgespeichert werden.

| Spalte  | Bedeutung   |
| ------- | ----------- |
| `r`     | Anzahl der Prozesse in `R` State (running or runnable) |
| `b`     | Anzahl der Prozesse in `D` State (uninteruptible sleep) |
| `swpd`  | Menge des benutzen virtuellen Speichers |
| `free`  | Menge des frei benutzbaren physikalischen Speichers |
| `buff`  | Menge an Speicher für Buffer benutzt |
| `cache` | Menge an Speicher für Cache Objekte benutzt |
| `si`    | Menge an virtuellem Speicher swapped-in pro Sekunde |
| `so`    | Menge an virtuellem Sepicher swapped-out pro Sekunde |
| `bi`    | Blöcke/s gelesen von Blockgeräten |
| `bo`    | Blöcke/s geschrieben in Blockgeräte |
| `in`    | Interrupts/s |
| `cs`    | Contextswitches/s |
| `us`    | Prozent CPU in user mode |
| `sy`    | Prozent CPU in kernel mode |
| `id`    | Prozent CPU idle |
| `wa` (ab Linux 2.5.41) | Prozent CPU auf IO wartend |
| `st`    | Prozent CPU von einer VM gestohlen |

Benutzung:
`vmstat <zyklus in sec.> [<count>]`

```
dude@rechner:~$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 12692592 174584 2304784    0    0    33    47   75  111  0  0 99  0  0
 0  0      0 12692592 174584 2304784    0    0     0     0 1650 2772  0  0 100  0  0
 2  0      0 12692592 174584 2304784    0    0     0     0 4757 8304  2  1 98  0  0
 0  0      0 12692592 174584 2304784    0    0     0    20 1214 2181  1  0 99  0  0
```

#### Per-Core Auslastung loggen
`mpstat -P ALL` loggt zyklisch verschiedenen CPU-Auslastungswerte per Core.

Benutzung:
`mpstat -P ALL [<andere options>] <zyklus in sec.> [<count>]`

```
dude@rechner:~$ mpstat -P ALL 1
Linux 5.4.0-72-generic (rechner) 	11.05.2021 	_x86_64_	(8 CPU)

07:43:23     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:43:24     all    0,50    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,50
07:43:24       0    0,99    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,01
07:43:24       1    0,99    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,01
07:43:24       2    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00  100,00
07:43:24       3    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00  100,00
07:43:24       4    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00  100,00
07:43:24       5    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00  100,00
07:43:24       6    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00  100,00
07:43:24       7    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00  100,00

07:43:24     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:43:25     all    0,74    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,26
07:43:25       0    0,99    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,01
07:43:25       1    0,98    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,02
07:43:25       2    0,98    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,02
07:43:25       3    0,98    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,02
07:43:25       4    0,99    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,01
07:43:25       5    0,98    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,02
07:43:25       6    1,96    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   98,04
07:43:25       7    0,99    0,00    0,00    0,00    0,00    0,00    0,00    0,00    0,00   99,01
```

#### Fehlender Parallelismus
Problem:

Berichtet `mpstat` konstant eine hohe Auslastung von wenigen oder gar
nur einem Core könnte ein Prozess der stark Singlethreaded ist daran Schuld sein.

Analyse:

Mit `htop` den verursachenden Prozess finden.


#### Hohe %IRQ-Auslastung
Problem:

Eine hohe %IRQ-Auslastung bei `mpstat` könnte auf ein Problem mit
Gerätetreibern, Memory-swapping oder anderen Dingen hinweisen.

Analyse:

`cat /proc/interrupts` zeigt die Anzahl der jeweils aufegetretenen Interrupts per Core an.
Damit kann man die verursachende Quelle finden.

#### Fine-grained Profiling mit perf
_todo_


### Memory
_todo_

### Disk IO
_todo_

### Network IO
_todo_
