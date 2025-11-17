
# Importación de bases de datos de disponibilidad IAG e IAGDAEMON

## Stop BALoader AWS (Destino)
## Stop BALoader Telefonica (Origen)

## Waiting exports telefonica (Origen)

## Start BALoader Telefonica (Origen)
## Get from database server gz dump files and send to AWS S3
## Acceder al servidor ftp de LIVE a la carpeta /mnt/data/exports_live/IAGPRDB y ejecutar el comando aws s3 cp a los 3 buckets de PRE, PRO e INT cargando las variables previamente.

## Copy dump to RDS

### Int desde sqldeveloper del bastion
```
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
p_bucket_name => '654654216237-int-exportdb',
p_s3_prefix => 'oracle/imports/exp_iagprcdb_20250601.dmp.gz',
p_directory_name => 'DATA_PUMP_DIR',
p_decompression_format => 'GZIP',
p_error_on_zero_downloads => 'TRUE')
AS TASK_ID FROM DUAL; 
```

### Pre desde sqldeveloper del bastion esquema admin
```
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
p_bucket_name => '767397981209-pre-exportdb',
p_s3_prefix => 'oracle/imports/exp_iagprcdb_20250601.dmp.gz',
p_directory_name => 'DATA_PUMP_DIR',
p_decompression_format => 'GZIP',
p_error_on_zero_downloads => 'TRUE')
AS TASK_ID FROM DUAL; 
```

### Prod desde sqldeveloper del bastion esquema admin
```
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
p_bucket_name => '471112533577-prod-exportdb',
p_s3_prefix => 'oracle/imports/exp_iagprcdb_20250601.dmp.gz',
p_directory_name => 'DATA_PUMP_DIR',
p_decompression_format => 'GZIP',
p_error_on_zero_downloads => 'TRUE')
AS TASK_ID FROM DUAL; 
```

## Verificar que la copia se ha completado esquema admin
```
SELECT * FROM TABLE(rdsadmin.rds_file_util.listdir('DATA_PUMP_DIR')) ORDER BY MTIME;
```

## Import de BBDD

### Realizar import de usuario iag desde el esquema admin
```
DECLARE
  v_hdnl NUMBER;
BEGIN
  v_hdnl := DBMS_DATAPUMP.OPEN( 
    operation => 'IMPORT', 
    job_mode  => 'SCHEMA', 
    job_name  => null);
  DBMS_DATAPUMP.ADD_FILE( 
    handle    => v_hdnl, 
    filename  => 'exp_iagprcdb_20250601.dmp', 
    directory => 'DATA_PUMP_DIR', 
    filetype  => dbms_datapump.ku$_file_type_dump_file);
  DBMS_DATAPUMP.ADD_FILE( 
    handle    => v_hdnl, 
    filename  => 'iagba_import.log', 
    directory => 'DATA_PUMP_DIR', 
    filetype  => dbms_datapump.ku$_file_type_log_file);
  DBMS_DATAPUMP.METADATA_FILTER(v_hdnl,'SCHEMA_EXPR','IN (''IAGBA'')');
  DBMS_DATAPUMP.SET_PARAMETER(handle => v_hdnl, name => 'TABLE_EXISTS_ACTION', value => 'REPLACE');
  DBMS_DATAPUMP.START_JOB(v_hdnl);
END;
/
```

## Verificar que el import se ha completado esquema admin
```
SELECT * FROM TABLE
(rdsadmin.rds_file_util.read_text_file(
p_directory => 'DATA_PUMP_DIR',
p_filename  => 'iagba_import.log'));
```

## Realizar import de usuario iagdaemonba desde el esquema admin
```
DECLARE
  v_hdnl NUMBER;
BEGIN
  v_hdnl := DBMS_DATAPUMP.OPEN( 
    operation => 'IMPORT', 
    job_mode  => 'SCHEMA', 
    job_name  => null);
  DBMS_DATAPUMP.ADD_FILE( 
    handle    => v_hdnl, 
    filename  => 'exp_iagprcdb_20250601.dmp', 
    directory => 'DATA_PUMP_DIR', 
    filetype  => dbms_datapump.ku$_file_type_dump_file);
  DBMS_DATAPUMP.ADD_FILE( 
    handle    => v_hdnl, 
    filename  => 'iagbadaemon_import.log', 
    directory => 'DATA_PUMP_DIR', 
    filetype  => dbms_datapump.ku$_file_type_log_file);
  DBMS_DATAPUMP.METADATA_FILTER(v_hdnl,'SCHEMA_EXPR','IN (''IAGDAEMONBA'')');
  DBMS_DATAPUMP.SET_PARAMETER(handle => v_hdnl, name => 'TABLE_EXISTS_ACTION', value => 'REPLACE');
  DBMS_DATAPUMP.START_JOB(v_hdnl);
END;
/
```

### Verificar que el import se ha completado desde el esquema admin
```
SELECT * FROM TABLE
(rdsadmin.rds_file_util.read_text_file(
p_directory => 'DATA_PUMP_DIR',
p_filename  => 'iagbadaemon_import.log'));
```

### Truncar tablas de iagdaemonba desde el esquema de iagdaemonba

```
truncate table MCP_SINGLE_OBJECT;
truncate table FLIGHT;
truncate table AIRPORT;
truncate table IBAIRPORT;
truncate table TFLTPOSMAPPINGS;
truncate table TBEPOSMAPPING;
truncate table IBMAPPINGS;

```

### Create additional NAVs tables (unused) on IAGBA SCHEMA

```
CREATE TABLE "IAGBA"."BA_NAVS" 
   (	"AIRLINE_CODE" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"FLIGHT_NUMBER" VARCHAR2(4 BYTE) NOT NULL ENABLE, 
	"DEPARTURE" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"ARRIVAL" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"DEPARTURE_DATE" DATE NOT NULL ENABLE, 
	"RBD" VARCHAR2(1 BYTE) NOT NULL ENABLE, 
	"SUB_CLASS" NUMBER(2,0) NOT NULL ENABLE, 
	"STATUS" VARCHAR2(1 BYTE) NOT NULL ENABLE
   ); 

CREATE TABLE "IAGBA"."BA_NAVS2" 
   (	"AIRLINE_CODE" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"FLIGHT_NUMBER" VARCHAR2(4 BYTE) NOT NULL ENABLE, 
	"DEPARTURE" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"ARRIVAL" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"DEPARTURE_DATE" DATE NOT NULL ENABLE, 
	"RBD" VARCHAR2(3 BYTE) NOT NULL ENABLE, 
	"AVAILABILITY" VARCHAR2(150 BYTE) NOT NULL ENABLE, 
	 CONSTRAINT "BA_NAVS2_PK" PRIMARY KEY ("AIRLINE_CODE", "FLIGHT_NUMBER", "DEPARTURE", "ARRIVAL", "DEPARTURE_DATE", "RBD"));

```

## Send all tables to daemon [load_data_iagdaemonba.ps1](https://multinucleo.visualstudio.com/IAG%20PHASE%201/_git/awsscripts?path=/windows/loads_data_iagdaemonba.ps1)

## Start BALoader AWS


