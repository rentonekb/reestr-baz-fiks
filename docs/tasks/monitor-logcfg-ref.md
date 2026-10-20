# logcfg.xml, сгенерированный 1С-Монитором (MSK-1C8-APP01, myway), снят 20.10.2026

Файл `\\MSK-1C8-APP01\c$\Program Files\1cv8\conf\logcfg.xml`, изменён 25.08.2026, 14 КБ. Пишет сам Монитор через агента.

Принцип: на каждую базу отдельные `<log>` со своим namespace-префиксом (`QR_`/`LK_`/`DL_`/`EX_` + `<база>_<порт>`),
каталог `D:\logs_1c\MonitorLogs\<база>_<порт>\<Вид>`, history 1–2 ч, фильтр `p:processName = <база>`, только нужные свойства.

| Каталог | События | Фильтр | Свойства |
|---|---|---|---|
| Queries | DBMSSQL | duration >= 50000 (5 с в 1/10000) | processName, clientID, applicationName, computerName, connectID, Context, Usr, dbpid, Sql, trans, Rows, RowsAffected, DataBase, DBCopy |
| Transactions | SDBL func=CommitTransaction / RollbackTransaction | — | processName, clientID, connectID, Usr, lka, Func |
| 1C_Locks | TLOCK, TTIMEOUT | **без порога длительности** | processName, clientID, applicationName, computerName, connectID, Context, Usr, Locks, escalating, WaitConnections |
| SQL_Locks | DBMSSQL like `UPDATE% dbo._%`, `DELETE% dbo._%`, `INSERT INTO dbo._%` | — | processName, computerName, connectID, applicationName, Context, Usr, dbpid, Sql, lkp, lksrc, clientID |
| (dbmslocks) | `<dbmslocks/>` | — | сбор блокировок СУБД платформой |
| 1C_Deadlocks | TDEADLOCK | — | processName, computerName, connectID, Context, Usr, DeadlockConnectionIntersections |
| Exceptions | EXCP (exception != 9db1fa37-…, «мусорное» исключение), SDBL Func=setRollbackOnly, QERR | — | all |

Служебные: `Processes_Performance` — CLSTR event="Performance update" (Data, computerName), history 1; `dump` в D:\logs_1c\Dumps.

Почему у нас (88.90, tj_perf) взаимоблокировки не разбираются:
- TLOCK пишется только >= 1 с — без всех TLOCK нет цепочки «кто держал»;
- нет транзакций SDBL Commit/Rollback (границы транзакций, чтобы найти виновника);
- нет `<dbmslocks/>` и записей DML для блокировок СУБД;
- нет свойства DeadlockConnectionIntersections в отдельном логе (у нас property all — оно есть, но смешано).
Схема Монитора: TDEADLOCK.DeadlockConnectionIntersections даёт пары соединений и пространства → по connectID берутся TLOCK (Locks, WaitConnections) и транзакции обеих сторон → Context обеих сторон.
