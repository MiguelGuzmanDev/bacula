# Manual 

Agregar nuevo cliente a Bacula.

## Configuración

### Configurar cliente remoto

```bash
# Instalar Cliente:
yum install -y bacula-client policycoreutils-python-utils

# Firewalld
firewall-cmd --add-service=bacula-client --permanent

firewall-cmd --reload

firewall-cmd --list-all

# Conf de cliente:
cat << EOF > /etc/bacula/bacula-fd.conf
Director {
  Name = bacula-dir
  Password = "${password}"
}

FileDaemon {
  Name = ${name_client}
  WorkingDirectory = /var/spool/bacula
  Pid Directory = /var/run
  Maximum Concurrent Jobs = 10
  Plugin Directory = /usr/lib64/bacula
}

Messages {
  Name = Standard
  director = bacula-dir = all, !skipped, !restored
}

EOF

# Crear directorios:
mkdir -p /var/backups/bacula /var/backups/restores

# Permisos:
find /var/backups -type d -exec chmod 2770 {} \;
find /var/backups -type f -exec chmod 660 {} \;

chgrp -R root /var/backups/
chown -R root /var/backups/

# SElinux:
semanage fcontext -a '/var/backups(/.*)?' -t bacula_store_t
restorecon -Rv /var/backups

##############################################################################
# En caso de ejecutar scripts los configuramos:
vi /usr/local/libexec/create_backup.sh

vi /usr/local/libexec/delete_backup.sh

# Permisos:
chmod 500 /usr/local/libexec/create_backup.sh
chmod 500 /usr/local/libexec/delete_backup.sh
chgrp bacula /usr/local/libexec/create_backup.sh
chgrp bacula /usr/local/libexec/delete_backup.sh
##############################################################################

# Habilitamos bacula client
systemctl enable --now bacula-fd.service

# Validamos todo Ok
systemctl status bacula-fd.service
```

### Configurar cliente local

#### Client

```bash
# Configuración:
# Generar password:
apg -M CLN -m 30 -n 1

cat << EOF > /etc/bacula/clients/cdmxv-rhvm.nube.gob.mx.conf
Client {
  Name = cdmxv-rhvm.nube.gob.mx-fd
  Catalog = MyCatalog
  Address = 172.30.0.13
  Password = "DecItweesIcEOjbiugVeckVuabram5"
  File Retention = 60 days
  Job Retention = 6 months
  AutoPrune = yes
}

EOF
```

#### Fileset

```bash
# Configuración;
cat << EOF > /etc/bacula/filesets/rhvm.conf
FileSet {
  Name = "FS-rhvm-Set"
  Include {
    Options {
      signature = MD5
	  xattrsupport = yes
	  aclsupport = yes
    }

	File = "/var/backups/bacula"
  }
}

EOF
```

#### Job

```bash
# Configuración;
cat << EOF > /etc/bacula/jobs/cdmxv-rhvm.nube.gob.mx.conf
Job {
  Name = "backup-cdmxv-rhvm.nube.gob.mx"
  Client = "cdmxv-rhvm.nube.gob.mx-fd"
  JobDefs = "DefaultJob"
  FileSet = "FS-rhvm-Set"
  ClientRunBeforeJob = "/usr/local/libexec/create_backup.sh"
  ClientRunAfterJob = "/usr/local/libexec/delete_backup.sh"
}

EOF
```

### Bacula

```bash
# Validamos configuración:
bacula-dir -c /etc/bacula/bacula-dir.conf -t

# Bconsole
bconsole

reload

status client=cdmxv-main-rhvm.nube.gob.mx-fd

run job=backup-cdmxv-main-rhvm.nube.gob.mx yes
```