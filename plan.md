
![Tool Overview](http://www.brendangregg.com/Perf/linux_observability_tools.png)

## Tools
* `htop`
* `mpstat`, `vmstat`, `iostat` aus dem Package `sysstat`
* `mmc` aus dem Package `mmc-utils`

## Probleme & Lösungen

### Index
* [Vollständigen Systemüberblick erhalten]
* [Überblick loggen]
* CPU
  * [Per-Core Auslastung loggen]
  * [Fehlender Parallelismus]
  * [Hohe %IRQ-Auslastung]
  * [Fine-grained Profiling mit perf]
* Memory
  * [Überschüssige Swaps]
  * [Memory-leak bemerken]
  * [Memory-leak debuggen mit Valgrind]
* Disk IO
* Network IO


### Vollständigen Systemüberblick erhalten
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

### Überblick loggen
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

### CPU

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

#### Überschüssige Swaps
Ein nah voller physikalischer Speicher kann sich stark negativ auf die Performance auswirken.

Die Spalten `MAJFLT` (Major Fault) sowie `CMAJFLT` (Major Faults of Child Processes) in `htop`
zeigen die Anzahl der Page Faults die das laden einer swapped-out Page zur Folge hatten.
Analog zeigen `MINFLT` und `CMINFLT` die Anzahl der Page Faults ohne swap-in, hauptsächlich
verursacht durch lazy Memory-allocation. Letztere sind jedoch generell vernachlässigbar.

```
  1  [                                              0.0%]   5  [                                              0.0%]
  2  [                                              0.0%]   6  [                                              0.0%]
  3  [                                              0.0%]   7  [                                              0.0%]
  4  [                                              0.0%]   8  [                                              0.0%]
  Mem[|||||||||||                            1.18G/15.6G]   Tasks: 141, 381 thr; 1 running
  Swp[                                           0K/472M]   Load average: 0.07 0.06 0.10 
                                                            Uptime: 00:23:27

     MAJFLT     CMAJFLT   PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
         86        7926     1 root       20   0  220M  9464  6636 S  0.0  0.1  0:04.32 /sbin/init splash
        155           0   297 root       19  -1  101M 26336 25236 S  0.0  0.2  0:00.68 /lib/systemd/systemd-journald
          5         265   316 root       20   0 34720  4600  2872 S  0.0  0.0  0:00.28 /lib/systemd/systemd-udevd
          4           0   627 systemd-r  20   0 70784  5480  4916 S  0.0  0.0  0:00.08 /lib/systemd/systemd-resolved
          1           0   666 root       20   0  4548   892   832 S  0.0  0.0  0:00.04 /usr/sbin/acpid
          9           0   667 messagebu  20   0 51716  6272  3992 S  0.0  0.0  0:01.44 /usr/bin/dbus-daemon --system --a
         24           0   749 root       20   0 45228  5268  4724 S  0.0  0.0  0:00.03 /sbin/wpa_supplicant -u -s -O /ru
         15          19   752 root       20   0  281M  7168  6208 S  0.0  0.0  0:00.23 /usr/lib/accountsservice/accounts
         62           0   753 root       20   0  617M 16876 13996 S  0.0  0.1  0:00.48 /usr/sbin/NetworkManager --no-dae
          5           0   760 avahi      20   0 47252  3208  2868 S  0.0  0.0  0:00.12 avahi-daemon: running [StudiumROS
         40           0   761 root       20   0  352M  9572  8160 S  0.0  0.1  0:00.11 /usr/sbin/ModemManager --filter-p
         28           4   763 root       20   0  491M 10796  8632 S  0.0  0.1  0:00.20 /usr/lib/udisks2/udisksd
         41           1   771 root       20   0  166M 17704  9752 S  0.0  0.1  0:00.32 /usr/bin/python3 /usr/bin/network
          2           0   774 root       20   0  107M  2060  1840 S  0.0  0.0  0:00.03 /usr/sbin/irqbalance --foreground
         10           0   775 syslog     20   0  256M  4540  3768 S  0.0  0.0  0:00.11 /usr/sbin/rsyslogd -n
          1           0   776 root       20   0 70756  6128  5312 S  0.0  0.0  0:00.31 /lib/systemd/systemd-logind
          1           0   778 root       20   0 31644  3236  2952 S  0.0  0.0  0:00.00 /usr/sbin/cron -f
          0           0   783 root       20   0  107M  2060  1840 S  0.0  0.0  0:00.00 /usr/sbin/irqbalance --foreground
          0          19   785 root       20   0  281M  7168  6208 S  0.0  0.0  0:00.16 /usr/lib/accountsservice/accounts
          0           0   786 avahi      20   0 47072   340     0 S  0.0  0.0  0:00.00 avahi-daemon: chroot helper
          0          19   795 root       20   0  281M  7168  6208 S  0.0  0.0  0:00.02 /usr/lib/accountsservice/accounts
          0           4   797 root       20   0  491M 10796  8632 S  0.0  0.1  0:00.00 /usr/lib/udisks2/udisksd
F1Help  F2Setup F3SearchF4FilterF5Tree  F6SortByF7Nice -F8Nice +F9Kill  F10Quit
```

#### Memory-leak bemerken
Will man wissen ob das System einen potentiellen Memory-leak hat kann man das Tool
`vmstat` benutzen.

`vmstat -tn 60 > ~/memleak.txt` loggt alle 60 Sekunden die Speicherauslastung mit Timestamp in die Datei `memleak.txt`.
Zeigt diese, dass die Speicherauslastung kontinuierlich heranwächst und wenn ein unerwartetes Abstürzen eines Programmes
zeitgleich mit einem deutlich leereren Speicher dahergeht könnte dies auf ein Speicherleck hinweisen,
das potentiell von jenem Programm verursacht wird.

```
dude@rechner:~$ vmstat -tn 60 > ~/memleak.txt
dude@rechner:~$ cat ~/memleak.txt
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----- -----timestamp-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st                CEST
 0  0      0 13120452 202140 1886436    0    0    70    63   83  129  1  0 99  0  0 2021-05-28 18:33:09
 0  0      0 13120452 202140 1886360    0    0     0     0 1532 2824  1  0 99  0  0 2021-05-28 18:34:09
 0  0      0 13120452 202140 1886360    0    0     0     0  238  435  0  0 100  0  0 2021-05-28 18:35:09
 1  0      0 13120452 202148 1886360    0    0     0    52 2303 4317  1  0 99  0  0 2021-05-28 18:36:09
 0  0      0 13120452 202148 1886360    0    0     0     0 2214 4117  1  0 99  0  0 2021-05-28 18:37:09
 0  0      0 13120452 202148 1886360    0    0     0     0  225  421  0  0 100  0  0 2021-05-28 18:38:09
 2  0      0 13120452 202148 1886360    0    0     0     0  157  270  0  0 100  0  0 2021-05-28 18:39:09
 0  0      0 13120452 202148 1886360    0    0     0     0  217  391  0  0 100  0  0 2021-05-28 18:40:09
 0  0      0 13120452 202148 1886360    0    0     0     0  226  388  0  0 100  0  0 2021-05-28 18:41:09
 0  0      0 13120452 202156 1886360    0    0     0    12  175  300  0  0 100  0  0 2021-05-28 18:42:09
 0  0      0 13120452 202156 1886360    0    0     0     0  256  477  0  0 100  0  0 2021-05-28 18:43:09
 0  0      0 13120452 202156 1886360    0    0     0     0  234  403  0  0 100  0  0 2021-05-28 18:44:09
 0  0      0 13120452 202156 1886360    0    0     0     0   81  114  0  0 100  0  0 2021-05-28 18:45:09
 0  0      0 13120452 202156 1886360    0    0     0     0   84  146  0  0 100  0  0 2021-05-28 18:46:09
 0  0      0 13120452 202156 1886360    0    0     0     0   81  133  0  0 100  0  0 2021-05-28 18:47:09
 0  0      0 13120452 202164 1886360    0    0     0    12   89  132  0  0 100  0  0 2021-05-28 18:48:09
 0  0      0 13120452 202164 1886360    0    0     0     0   70  119  0  0 100  0  0 2021-05-28 18:49:09
 0  0      0 13120452 202164 1886360    0    0     0     0  104  143  0  0 100  0  0 2021-05-28 18:50:09
 0  0      0 13120452 202164 1886360    0    0     0     0   78  125  0  0 100  0  0 2021-05-28 18:51:09
 0  0      0 13120452 202164 1886360    0    0     0     0  984 1973  0  0 100  0  0 2021-05-28 18:52:09
 0  0      0 13120452 202164 1886360    0    0     0     0   74  123  0  0 100  0  0 2021-05-28 18:53:09
```

#### Memory-leak debuggen mit Valgrind
_todo_


### Disk IO

#### Geräteauslastung loggen
Mit `iostat -x <interval>` lässt sich die Geräteauslastung von Platten und anderen Geräten periodisch loggen.

Fällt bei einem Gerät eine besonders hohe Lese- oder Schreibauslastung und hohe Latenzen gleichzeitig auf,
kann das ein gutes Indiz dafür sein dass das Gerät aus- oder überlastet ist.
Der `%util` Spalte kann man jedoch nicht trauen. Sie gibt lediglich die Verbrauchte CPU Zeit an was ausschließlich
bei sequentiellen Operationen akkurat ist.

| Spalte  | Bedeutung   |
| ------- | ----------- |
| `r/s`   | Anzahl an Lesebefehlen abgeschlossen pro Sekunde auf dem Gerät |
| `w/s`   | Anzahl an Schreibbefehlen abgeschlossen pro Sekunde auf dem Gerät |
| `rkB/s` | Datenmenge gelesen pro Sekunde von dem Gerät |
| `wkB/s` | Datenmenge geschrieben pro Sekunde in das Gerät |
| `rrqm/s`   | Anzahl an Lesebefehlen die gemerged wurden |
| `wrqm/s`   | Anzahl an Schreibbefehlen die gemerged wurden |
| `%rrqm` | Prozent der Lesebefehle die gemerged wurden vor dem Senden zum Gerät |
| `%wrqm` | Prozent der Schreibbefehle die gemerged wurden vor dem Senden zum Gerät |
| `r_await`  | Durchschnittliche Latenz in Millisekunden von Lesebefehlen |
| `w_await`  | Durchschnittliche Latenz in Millisekunden von Schreibbefehlen |
| `aqu-sz`   | Durschnittliche Queue-länge der Befehle die an das Gerät gesendet wurden |
| `rareq-sz` | Durschnittliche Größe in KB von Lesebefehlen die an das Gerät gesendet wurden |
| `wareq-sz` | Durschnittliche Größe in KB von Schreibbefehlen die an das Gerät gesendet wurden |
| `svctm` | _Deprecated_ |
| `%util` | Prozent Zeit die benutzt wird um Befehle an das Gerät zu senden (irreführender Wert) |

**Beispiel:**
```
dude@rechner:~$ iostat -x 1
Linux 5.4.0-73-generic (rechner) 	14.06.2021 	_x86_64_	(8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0,68    0,15    0,27    0,04    0,00   98,87

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
loop0            0,09    0,00      0,20      0,00     0,00     0,00   0,00   0,00    0,18    0,00   0,00     2,38     0,00   0,32   0,00
loop1            0,22    0,00      0,62      0,00     0,00     0,00   0,00   0,00    0,19    0,00   0,00     2,86     0,00   0,25   0,01
loop2            0,02    0,00      0,05      0,00     0,00     0,00   0,00   0,00    0,83    0,00   0,00     2,63     0,00   1,13   0,00
loop3            0,01    0,00      0,02      0,00     0,00     0,00   0,00   0,00    1,68    0,00   0,00     2,42     0,00   2,32   0,00
loop4            0,01    0,00      0,02      0,00     0,00     0,00   0,00   0,00    1,00    0,00   0,00     2,59     0,00   1,65   0,00
loop5            0,02    0,00      0,05      0,00     0,00     0,00   0,00   0,00    0,76    0,00   0,00     2,71     0,00   1,14   0,00
loop6            0,01    0,00      0,02      0,00     0,00     0,00   0,00   0,00    0,78    0,00   0,00     2,50     0,00   1,33   0,00
loop7            0,08    0,00      0,20      0,00     0,00     0,00   0,00   0,00    0,23    0,00   0,00     2,40     0,00   0,40   0,00
sda             21,17   25,50    575,60    804,30    12,55    24,12  37,22  48,61    0,48    0,25   0,01    27,18    31,54   0,31   1,42
loop8            4,26    0,00      4,67      0,00     0,00     0,00   0,00   0,00    1,34    0,00   0,00     1,10     0,00   0,15   0,06
loop9            0,21    0,00      0,61      0,00     0,00     0,00   0,00   0,00    0,28    0,00   0,00     2,94     0,00   0,32   0,01
loop10           0,04    0,00      0,44      0,00     0,00     0,00   0,00   0,00    1,06    0,00   0,00    11,67     0,00   1,06   0,00
loop11           0,02    0,00      0,05      0,00     0,00     0,00   0,00   0,00    1,46    0,00   0,00     2,63     0,00   1,48   0,00
loop12           0,03    0,00      0,43      0,00     0,00     0,00   0,00   0,00    1,61    0,00   0,00    14,55     0,00   1,03   0,00
loop13           0,05    0,00      0,47      0,00     0,00     0,00   0,00   0,00    0,60    0,00   0,00    10,02     0,00   0,76   0,00
loop14           0,09    0,00      0,50      0,00     0,00     0,00   0,00   0,00    0,18    0,00   0,00     5,41     0,00   0,40   0,00
loop15           0,07    0,00      0,47      0,00     0,00     0,00   0,00   0,00    4,39    0,00   0,00     7,24     0,00   2,55   0,02
loop16           0,01    0,00      0,02      0,00     0,00     0,00   0,00   0,00    8,47    0,00   0,00     2,42     0,00   6,11   0,00
loop17           0,02    0,00      0,05      0,00     0,00     0,00   0,00   0,00    1,08    0,00   0,00     3,05     0,00   1,37   0,00
loop18           0,00    0,00      0,00      0,00     0,00     0,00   0,00   0,00    0,00    0,00   0,00     1,00     0,00   1,00   0,00
```

#### Prozess mit hoher Disk-Auslastung finden
Um einen Prozess mit hoher Auslastung zu finden bieten sich in `htop` die Spalten
`IO_READ_RATE` und `IO_WRITE_RATE` an.
Diese zeigen den IO-Durchgang des Prozesses in bytes pro Sekunde an.
Der IO-Durchgang misst alle Geräte inklusive Blockgeräten und Netzwerkadapter.

```
  1  [                                                           0.0%]   5  [                                                           0.0%]
  2  [                                                           0.0%]   6  [                                                           0.0%]
  3  [                                                           0.0%]   7  [                                                           0.0%]
  4  [|                                                          0.7%]   8  [                                                           0.0%]
  Mem[||||||||||||||||                                    1.18G/15.6G]   Tasks: 144, 383 thr; 1 running
  Swp[                                                        0K/472M]   Load average: 0.31 0.12 0.08 
                                                                         Uptime: 01:09:59

  DISK READ  DISK WRITE      MAJFLT     CMAJFLT   PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
   4.30 B/s    1.20 B/s          86        7230     1 root       20   0  220M  9564  6724 S  0.0  0.1  0:03.30 /sbin/init splash
   0.00 B/s    0.00 B/s         101           0   299 root       19  -1  109M 14628 13488 S  0.0  0.1  0:00.30 /lib/systemd/systemd-journald
   0.00 B/s    0.00 B/s           5         256   317 root       20   0 34628  4748  2924 S  0.0  0.0  0:00.17 /lib/systemd/systemd-udevd
   0.00 B/s    0.00 B/s           4           0   513 systemd-r  20   0 70784  6228  5536 S  0.0  0.0  0:00.07 /lib/systemd/systemd-resolved
   0.00 B/s    0.00 B/s           6           0   693 avahi      20   0 47252  3320  2976 S  0.0  0.0  0:00.05 avahi-daemon: running [StudiumROS.l
   0.00 B/s    0.00 B/s           0           1   880 root       20   0  166M 17516  9568 S  0.0  0.1  0:00.00 /usr/bin/python3 /usr/bin/networkd-
   0.00 B/s    0.00 B/s          44           1   700 root       20   0  166M 17516  9568 S  0.0  0.1  0:00.11 /usr/bin/python3 /usr/bin/networkd-
   0.00 B/s    0.00 B/s           0          19   719 root       20   0  281M  6988  6032 S  0.0  0.0  0:00.08 /usr/lib/accountsservice/accounts-d
   0.00 B/s    0.00 B/s           0          19   757 root       20   0  281M  6988  6032 S  0.0  0.0  0:00.02 /usr/lib/accountsservice/accounts-d
   0.00 B/s    0.00 B/s          15          19   702 root       20   0  281M  6988  6032 S  0.0  0.0  0:00.14 /usr/lib/accountsservice/accounts-d
   0.00 B/s    0.00 B/s           0           0   740 root       20   0  424M  9588  8124 S  0.0  0.1  0:00.00 /usr/sbin/ModemManager --filter-pol
   0.00 B/s    0.00 B/s           0           0   756 root       20   0  424M  9588  8124 S  0.0  0.1  0:00.00 /usr/sbin/ModemManager --filter-pol
   0.00 B/s    0.00 B/s          40           0   705 root       20   0  424M  9588  8124 S  0.0  0.1  0:00.06 /usr/sbin/ModemManager --filter-pol
   0.00 B/s    0.00 B/s           0           0   714 root       20   0  107M  2044  1824 S  0.0  0.0  0:00.00 /usr/sbin/irqbalance --foreground
   0.00 B/s    0.00 B/s           3           0   708 root       20   0  107M  2044  1824 S  0.0  0.0  0:00.08 /usr/sbin/irqbalance --foreground
   0.00 B/s    0.00 B/s           3           0   712 messagebu  20   0 51804  6500  4132 S  0.0  0.0  0:01.00 /usr/bin/dbus-daemon --system --add
   0.00 B/s    0.00 B/s           0           0   718 avahi      20   0 47072   344     0 S  0.0  0.0  0:00.00 avahi-daemon: chroot helper
   0.00 B/s    0.00 B/s          24           0   758 root       20   0 45228  5300  4760 S  0.0  0.0  0:00.04 /sbin/wpa_supplicant -u -s -O /run/
   0.00 B/s    0.00 B/s           1           0   759 root       20   0 31644  3448  3152 S  0.0  0.0  0:00.00 /usr/sbin/cron -f
   0.00 B/s    0.00 B/s           0           4   805 root       20   0  491M 11140  8796 S  0.0  0.1  0:00.00 /usr/lib/udisks2/udisksd
   0.00 B/s    0.00 B/s           0           4   807 root       20   0  491M 11140  8796 S  0.0  0.1  0:00.00 /usr/lib/udisks2/udisksd
F1Help  F2Setup F3SearchF4FilterF5Tree  F6SortByF7Nice -F8Nice +F9Kill  F10Quit

```

#### SMART-Informationen auslesen (unterstützte Platten)
**Tool:** `smartctl` aus `smartmontools`

Auf Platten die SMART unterstützen lassen sich mit dem Tool `smartctl` Informationen auslesen.
Hohe IO-Latenzen oder schlechte Throughput-Werte aus `iostat` können an abgenutzten Flashspeicher liegen,
wenn die `Remaining` Spalte eine hohe Abnutzung zeigt.

Die SMART-Werte sind jedoch nur auf unterstützten Platten verfügbar und sind in manchen Fällen nicht 100% akkurat.

**Beispiel:**
```
dude@rechner:~$ sudo smartctl -a /dev/sda
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     17877         -
# 2  Extended offline    Completed without error       00%      8449         -
# 3  Short offline       Completed without error       00%      8446         -
# 4  Short offline       Completed without error       00%      1307         -
# 5  Short offline       Completed without error       00%         2         -
# 6  Extended offline    Self-test routine in progress 90%     17877         -
```

#### MMC-Informationen auslesen (unterstützte Geräte)
**Tool:** `mmc` aus `mmc-utils`

Mit dem Tool `mmc` lassen sich MMC-Informationen auslesen.

Wichtige Zeilen:
| Zeile | Bedeutung |
| ----- | --------- |
| `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A` | Ungefähr maximales verbrauchtes Leben der SLC Blöcke in 10% Schritten (`0x01` = 0%-10%, `0x02%` = 10%-20%, ..)|
| `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B` | Ungefähr maximales verbrauchtes Leben der MLC Blöcke |
| `EXT_CSD_PRE_EOL_INFO` | Status der reservierten Blöcke. (`0x01`: Normal, weniger als 80% der reservierten Blöcke in Benutzung. `0x02`: Warnung, 80% in Benutzung. `0x03`: Dringend, 90% in Benutzung) |

```
dude@rechner:~$ mmc extcsd read /dev/mmcblk0
=============================================
  Extended CSD rev 1.8 (MMC 5.1)
=============================================

Card Supported Command sets [S_CMD_SET: 0x01]
HPI Features [HPI_FEATURE: 0x01]: implementation based on CMD13
Background operations support [BKOPS_SUPPORT: 0x01]
Max Packet Read Cmd [MAX_PACKED_READS: 0x3c]
Max Packet Write Cmd [MAX_PACKED_WRITES: 0x3c]
Data TAG support [DATA_TAG_SUPPORT: 0x01]
Data TAG Unit Size [TAG_UNIT_SIZE: 0x03]
Tag Resources Size [TAG_RES_SIZE: 0x00]
Context Management Capabilities [CONTEXT_CAPABILITIES: 0x05]
Large Unit Size [LARGE_UNIT_SIZE_M1: 0x03]
Extended partition attribute support [EXT_SUPPORT: 0x03]
Generic CMD6 Timer [GENERIC_CMD6_TIME: 0x19]
Power off notification [POWER_OFF_LONG_TIME: 0xff]
Cache Size [CACHE_SIZE] is 512 KiB
Background operations status [BKOPS_STATUS: 0x00]
1st Initialisation Time after programmed sector [INI_TIMEOUT_AP: 0x64]
[...]
Native sector size [NATIVE_SECTOR_SIZE]: 0x00
Sector size emulation [USE_NATIVE_SECTOR]: 0x00
Sector size [DATA_SECTOR_SIZE]: 0x00
1st initialization after disabling sector size emulation [INI_TIMEOUT_EMU]: 0x00
Class 6 commands control [CLASS_6_CTRL]: 0x00
Number of addressed group to be Released[DYNCAP_NEEDED]: 0x00
Exception events control [EXCEPTION_EVENTS_CTRL]: 0x0000
Exception events status[EXCEPTION_EVENTS_STATUS]: 0x0000
Extended Partitions Attribute [EXT_PARTITIONS_ATTRIBUTE]: 0x0000
Context configuration [CONTEXT_CONF[51]]: 0x00
[...]
Packed command status [PACKED_COMMAND_STATUS]: 0x00
Packed command failure index [PACKED_FAILURE_INDEX]: 0x00
Power Off Notification [POWER_OFF_NOTIFICATION]: 0x01
Control to turn the Cache ON/OFF [CACHE_CTRL]: 0x01
eMMC Firmware Version:
eMMC Life Time Estimation A [EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A]: 0x01
eMMC Life Time Estimation B [EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B]: 0x01
eMMC Pre EOL information [EXT_CSD_PRE_EOL_INFO]: 0x01
Secure Removal Type [SECURE_REMOVAL_TYPE]: 0x01
 information is configured to be removed by an erase of the physical memory
 Supported Secure Removal Type:
  information removed by an erase of the physical memory
Command Queue Support [CMDQ_SUPPORT]: 0x01
Command Queue Depth [CMDQ_DEPTH]: 32
Command Enabled [CMDQ_MODE_EN]: 0x00
```


### Network IO
_todo_

