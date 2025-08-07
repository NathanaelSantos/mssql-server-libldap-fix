
````markdown
# mssql-server-libldap-fix


# Diagnosis and Fix for the `mssql-server.service` on Linux

---

## Initial Problem

When checking the status of the MSSQL Server service, it is failing:

```bash
$ systemctl status mssql-server.service --no-pager
× mssql-server.service - Microsoft SQL Server Database Engine
     Loaded: loaded (/usr/lib/systemd/system/mssql-server.service; enabled; preset: enabled)
     Active: failed (Result: exit-code) since Wed 2025-08-06 23:19:06 -03; 5min ago
   Duration: 4ms
       Docs: https://docs.microsoft.com/en-us/sql/linux
    Process: 620868 ExecStart=/opt/mssql/bin/sqlservr (code=exited, status=127)
   Main PID: 620868 (code=exited, status=127)
````

---

## Service Logs

No journal entries in the last 5 minutes:

```bash
$ journalctl -u mssql-server.service --since "5 minutes ago"
-- No entries --
```

---

## Attempt to start the service and inspect logs

```bash
$ systemctl start mssql-server.service --no-pager
$ journalctl -u mssql-server.service --since "5 minutes ago"
```

Logs indicate a missing library error:

```
/opt/mssql/bin/sqlservr: error while loading shared libraries: liblber-2.5.so.0: cannot open shared object file: No such file or directory
```

---

## Check for missing libraries

```bash
$ ldd /opt/mssql/bin/sqlservr | grep "not found"
	liblber-2.5.so.0 => not found
	libldap-2.5.so.0 => not found
```

---

## Download and install a compatible version of `libldap`

```bash
$ curl -O https://debian.mirror.ac.za/debian/pool/main/o/openldap/libldap-2.5-0_2.5.13+dfsg-5_amd64.deb
$ sudo dpkg -i libldap-2.5-0_2.5.13+dfsg-5_amd64.deb
```

Installation completed without issues.

---

## Restart MSSQL service

```bash
$ systemctl start mssql-server.service --no-pager
```

---

## Check MSSQL service status - Service active and running

```bash
$ systemctl status mssql-server.service --no-pager
● mssql-server.service - Microsoft SQL Server Database Engine
     Loaded: loaded (/usr/lib/systemd/system/mssql-server.service; enabled; preset: enabled)
     Active: active (running) since Wed 2025-08-06 23:41:02 -03; 30s ago
       Docs: https://docs.microsoft.com/en-us/sql/linux
   Main PID: 622997 (sqlservr)
      Tasks: 84
     Memory: 414.8M (peak: 415.1M)
        CPU: 10.358s
     CGroup: /system.slice/mssql-server.service
             ├─622997 /opt/mssql/bin/sqlservr
             └─623004 /opt/mssql/bin/sqlservr
```

---

# Conclusion

The initial failure of the Microsoft SQL Server service was caused by missing libraries `liblber-2.5.so.0` and `libldap-2.5.so.0`. Manually installing the correct version of the `libldap-2.5-0` package (compatible with system dependencies) resolved the problem, allowing the service to start and run correctly.

---

# References

* [Microsoft SQL Server on Linux documentation](https://docs.microsoft.com/en-us/sql/linux)
* System logs: `journalctl`
* Command to check libraries: `ldd`
