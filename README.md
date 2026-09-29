# Informe Final: Migración de Sistema Informático a PostgreSQL 18

**Asignatura:** Tecnología de Base de Datos I  
**Estudiante:** Juan Gabriel Ferrufino Vásquez  
**Docente:** Jared Lopez Leaño  
**Fecha de Entrega:** 29 de Septiembre, 2026  


---

## Resumen Ejecutivo

El presente informe detalla la migración de la base de datos `employees` desde MariaDB hacia PostgreSQL 18, sobre un entorno contenedorizado en Docker. Se aborda la transferencia de estructuras, datos y vistas, y se documentan las verificaciones de volumen, contenido e integridad referencial realizadas entre ambos motores. Durante la verificación se detectó que las tablas `employees` y `departments` habían quedado vacías y que la estructura destino carecía de llaves primarias y foráneas; ambos problemas se corrigieron y se documentan en las secciones 1.2 y 1.3.

---

## 1. Migración de Tablas (40 pts)


### 1.1 Metodología y comandos
También se comprobó inicialmente que las tablas se encontraban vacías:

```bash
psql -h localhost -U juan -d pdb_employees -c "SELECT 'departments' AS tabla, COUNT(*) FROM employees.departments UNION ALL SELECT 'employees', COUNT(*) FROM employees.employees UNION ALL SELECT 'dept_emp', COUNT(*) FROM employees.dept_emp UNION ALL SELECT 'dept_manager', COUNT(*) FROM employees.dept_manager UNION ALL SELECT 'salaries', COUNT(*) FROM employees.salaries UNION ALL SELECT 'titles', COUNT(*) FROM employees.titles;"
```

Salida:

```text
    tabla     | count
--------------+-------
 departments  |     0
 employees    |     0
 dept_emp     |     0
 dept_manager |     0
 salaries     |     0
 titles       |     0
(6 filas)
```

Esto confirmó que la estructura estaba preparada para recibir los datos.

---

Para la transferencia automatizada de estructura y datos se utilizó `pgloader`:

```bash
pgloader mysql://root:TU_PASSWORD@127.0.0.1:3306/employees?useSSL=false postgresql://juan:TU_PASSWORD@127.0.0.1:5433/pdb_employees
```

### 1.2 Hallazgo en la verificación y recarga de datos

Al comparar los conteos entre ambos motores se detectó una discrepancia: en PostgreSQL las tablas `departments` y `employees` tenían **0 filas**, mientras que en MariaDB tenían 9 y 300024 respectivamente. Las otras cuatro tablas sí coincidían.

```text
    tabla     |  filas  
--------------+---------
 departments  |       0
 dept_emp     |  331603
 dept_manager |      24
 employees    |       0
 salaries     | 2844047
 titles       |  443308
```

Se descartó que los datos estuvieran en otro esquema (no existe `public.employees`). Se recargaron únicamente esas dos tablas con `pgloader` en modo solo datos, mediante el archivo `recarga_datos.load`:

```lisp
LOAD DATABASE
     FROM mysql://root:TU_PASSWORD@127.0.0.1:3306/employees
     INTO postgresql://juan:TU_PASSWORD@127.0.0.1:5433/pdb_employees

 WITH data only,
      truncate,
      create no tables,
      create no indexes,
      workers = 4, concurrency = 1

 INCLUDING ONLY TABLE NAMES MATCHING 'departments', 'employees'
;
```

```bash
pgloader recarga_datos.load
```

Salida de la recarga:

```text
             table name     errors       rows      bytes      total time
-----------------------  ---------  ---------  ---------  --------------
        fetch meta data          0          2                     0.164s
      Drop Foreign Keys          0          0                     0.000s
               Truncate          0          2                     0.000s
-----------------------  ---------  ---------  ---------  --------------
    employees.employees          0     300024    13.2 MB          0.528s
  employees.departments          0          9     0.1 kB          0.060s
-----------------------  ---------  ---------  ---------  --------------
COPY Threads Completion          0          4                     0.528s
        Reset Sequences          0          0                     0.064s
    Create Foreign Keys          0          0                     0.000s
       Install Comments          0          0                     0.000s
-----------------------  ---------  ---------  ---------  --------------
      Total import time          ✓     300033    13.2 MB          0.592s
```

### 1.3 Llaves primarias y foráneas

La consulta a `pg_constraint` mostró que la estructura destino no tenía llaves foráneas, y al intentar crearlas PostgreSQL respondió que no existía una restricción única en las tablas referenciadas: tampoco había llaves primarias. Con los datos ya completos se agregaron primero las llaves primarias:

```sql
ALTER TABLE employees.employees    ADD PRIMARY KEY (emp_no);
ALTER TABLE employees.departments  ADD PRIMARY KEY (dept_no);
ALTER TABLE employees.dept_emp     ADD PRIMARY KEY (emp_no, dept_no);
ALTER TABLE employees.dept_manager ADD PRIMARY KEY (emp_no, dept_no);
ALTER TABLE employees.salaries     ADD PRIMARY KEY (emp_no, from_date);
ALTER TABLE employees.titles       ADD PRIMARY KEY (emp_no, title, from_date);
```

Y luego las llaves foráneas:

```sql
ALTER TABLE employees.dept_emp     ADD FOREIGN KEY (emp_no)  REFERENCES employees.employees(emp_no)     ON DELETE CASCADE;
ALTER TABLE employees.dept_emp     ADD FOREIGN KEY (dept_no) REFERENCES employees.departments(dept_no) ON DELETE CASCADE;
ALTER TABLE employees.dept_manager ADD FOREIGN KEY (emp_no)  REFERENCES employees.employees(emp_no)     ON DELETE CASCADE;
ALTER TABLE employees.dept_manager ADD FOREIGN KEY (dept_no) REFERENCES employees.departments(dept_no) ON DELETE CASCADE;
ALTER TABLE employees.salaries     ADD FOREIGN KEY (emp_no)  REFERENCES employees.employees(emp_no)     ON DELETE CASCADE;
ALTER TABLE employees.titles       ADD FOREIGN KEY (emp_no)  REFERENCES employees.employees(emp_no)     ON DELETE CASCADE;
```

Los seis comandos se ejecutaron sin errores. Como PostgreSQL valida las filas existentes al crear una llave foránea, esto es una primera confirmación de que no hay registros huérfanos. Estado final de las restricciones:

```text
    tabla     |          conname          | contype | referencia  
--------------+---------------------------+---------+-------------
 departments  | departments_pkey          | p       | -
 dept_emp     | dept_emp_pkey             | p       | -
 dept_manager | dept_manager_pkey         | p       | -
 employees    | employees_pkey            | p       | -
 salaries     | salaries_pkey             | p       | -
 titles       | titles_pkey               | p       | -
 dept_emp     | dept_emp_emp_no_fkey      | f       | employees
 dept_emp     | dept_emp_dept_no_fkey     | f       | departments
 dept_manager | dept_manager_emp_no_fkey  | f       | employees
 dept_manager | dept_manager_dept_no_fkey | f       | departments
 salaries     | salaries_emp_no_fkey      | f       | employees
 titles       | titles_emp_no_fkey        | f       | employees
(12 filas)
```

### 1.4 Verificación de conteo de filas

Consulta ejecutada en ambos motores (en PostgreSQL con el prefijo `employees.` en cada tabla):

```sql
SELECT 'departments' AS tabla, COUNT(*) AS filas FROM departments
UNION ALL SELECT 'dept_emp', COUNT(*) FROM dept_emp
UNION ALL SELECT 'dept_manager', COUNT(*) FROM dept_manager
UNION ALL SELECT 'employees', COUNT(*) FROM employees
UNION ALL SELECT 'salaries', COUNT(*) FROM salaries
UNION ALL SELECT 'titles', COUNT(*) FROM titles;
```

**MariaDB:**

```text
tabla           filas
departments     9
dept_emp        331603
dept_manager    24
employees       300024
salaries        2844047
titles          443308
```

**PostgreSQL 18 (después de la recarga):**

```text
    tabla     |  filas  
--------------+---------
 departments  |       9
 dept_emp     |  331603
 dept_manager |      24
 employees    |  300024
 salaries     | 2844047
 titles       |  443308
(6 filas)
```

| Tabla | Registros MariaDB | Registros PostgreSQL 18 | Estado |
| --- | --- | --- | --- |
| `employees` | 300,024 | 300,024 | Coincide |
| `departments` | 9 | 9 | Coincide |
| `dept_emp` | 331,603 | 331,603 | Coincide |
| `dept_manager` | 24 | 24 | Coincide |
| `salaries` | 2,844,047 | 2,844,047 | Coincide |
| `titles` | 443,308 | 443,308 | Coincide |

---

## 2. Migración de Vistas (20 pts)

La base original tiene dos vistas: `dept_emp_latest_date` y `current_dept_emp`. La segunda depende de la primera, por lo que deben crearse en ese orden.

### 2.1 Definiciones originales (MariaDB)

```sql
CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%` SQL SECURITY DEFINER VIEW `dept_emp_latest_date` AS
select `dept_emp`.`emp_no` AS `emp_no`,
       max(`dept_emp`.`from_date`) AS `from_date`,
       max(`dept_emp`.`to_date`) AS `to_date`
from `dept_emp`
group by `dept_emp`.`emp_no`;
```

```sql
CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%` SQL SECURITY DEFINER VIEW `current_dept_emp` AS
select `l`.`emp_no` AS `emp_no`, `d`.`dept_no` AS `dept_no`,
       `l`.`from_date` AS `from_date`, `l`.`to_date` AS `to_date`
from (`dept_emp` `d` join `dept_emp_latest_date` `l`
      on(`d`.`emp_no` = `l`.`emp_no` and `d`.`from_date` = `l`.`from_date` and `l`.`to_date` = `d`.`to_date`));
```

### 2.2 Adaptación DDL a PostgreSQL 18

Se eliminaron los apóstrofes invertidos (backticks), las cláusulas propias de MariaDB (`ALGORITHM`, `DEFINER`, `SQL SECURITY`) y los alias redundantes, manteniendo la lógica de agregación y unión:

```sql
CREATE OR REPLACE VIEW employees.dept_emp_latest_date AS
SELECT emp_no, MAX(from_date) AS from_date, MAX(to_date) AS to_date
FROM employees.dept_emp
GROUP BY emp_no;

CREATE OR REPLACE VIEW employees.current_dept_emp AS
SELECT l.emp_no, d.dept_no, l.from_date, l.to_date
FROM employees.dept_emp d
JOIN employees.dept_emp_latest_date l
  ON d.emp_no = l.emp_no AND d.from_date = l.from_date AND l.to_date = d.to_date;
```

Además se construyó la vista complementaria `v_full_employees`, que no existe en la base original, usando el operador estándar de concatenación `||`:

```sql
CREATE OR REPLACE VIEW employees.v_full_employees AS
SELECT e.emp_no, e.first_name || ' ' || e.last_name AS full_name, d.dept_name
FROM employees.employees e
JOIN employees.dept_emp de ON e.emp_no = de.emp_no
JOIN employees.departments d ON de.dept_no = d.dept_no;
```

### 2.3 Verificación de las vistas

**Vistas existentes en PostgreSQL** (`\dv employees.*`):

```text
                Listado de vistas
  Esquema  |        Nombre        | Tipo  | Dueño 
-----------+----------------------+-------+-------
 employees | current_dept_emp     | vista | juan
 employees | dept_emp_latest_date | vista | juan
 employees | v_full_employees     | vista | juan
(3 filas)
```

**Comparación entre motores:**

| Prueba | MariaDB | PostgreSQL 18 |
| --- | --- | --- |
| `SELECT COUNT(*) FROM dept_emp_latest_date` | 300024 | 300024 |
| `SELECT COUNT(*) FROM current_dept_emp` | 300024 | 300024 |

Muestra de `current_dept_emp` ordenada por `emp_no` (`ORDER BY emp_no LIMIT 5`), idéntica en ambos motores:

```text
 emp_no | dept_no | from_date  |  to_date   
--------+---------+------------+------------
  10001 | d005    | 1986-06-26 | 9999-01-01
  10002 | d007    | 1996-08-03 | 9999-01-01
  10003 | d004    | 1995-12-03 | 9999-01-01
  10004 | d004    | 1986-12-01 | 9999-01-01
  10005 | d003    | 1989-09-12 | 9999-01-01
```

**Vista complementaria `v_full_employees`** (`ORDER BY emp_no LIMIT 10`):

```text
 emp_no |     full_name      |     dept_name      
--------+--------------------+--------------------
  10001 | Georgi Facello     | Development
  10002 | Bezalel Simmel     | Sales
  10003 | Parto Bamford      | Production
  10004 | Chirstian Koblick  | Production
  10005 | Kyoichi Maliniak   | Human Resources
  10006 | Anneke Preusig     | Development
  10007 | Tzvetan Zielinski  | Research
  10008 | Saniya Kalloufi    | Development
  10009 | Sumant Peac        | Quality Management
  10010 | Duangkaew Piveteau | Production
(10 filas)
```

Esta vista une empleados con sus asignaciones en `dept_emp`, por lo que un empleado que haya pertenecido a más de un departamento aparece una vez por cada asignación.

---

## 3. Consultas de Verificación e Integridad (40 pts)

### 3.1 Integridad referencial (registros huérfanos)

Se buscaron claves foráneas sin correspondencia en las seis relaciones del modelo:

```sql
SELECT 'dept_emp→employees' AS relacion, COUNT(*) AS huerfanos
FROM employees.dept_emp de LEFT JOIN employees.employees e ON de.emp_no=e.emp_no WHERE e.emp_no IS NULL
UNION ALL SELECT 'dept_emp→departments', COUNT(*)
FROM employees.dept_emp de LEFT JOIN employees.departments d ON de.dept_no=d.dept_no WHERE d.dept_no IS NULL
UNION ALL SELECT 'dept_manager→employees', COUNT(*)
FROM employees.dept_manager dm LEFT JOIN employees.employees e ON dm.emp_no=e.emp_no WHERE e.emp_no IS NULL
UNION ALL SELECT 'dept_manager→departments', COUNT(*)
FROM employees.dept_manager dm LEFT JOIN employees.departments d ON dm.dept_no=d.dept_no WHERE d.dept_no IS NULL
UNION ALL SELECT 'salaries→employees', COUNT(*)
FROM employees.salaries s LEFT JOIN employees.employees e ON s.emp_no=e.emp_no WHERE e.emp_no IS NULL
UNION ALL SELECT 'titles→employees', COUNT(*)
FROM employees.titles t LEFT JOIN employees.employees e ON t.emp_no=e.emp_no WHERE e.emp_no IS NULL;
```

```text
         relacion         | huerfanos 
--------------------------+-----------
 dept_emp→employees       |         0
 dept_emp→departments     |         0
 dept_manager→employees   |         0
 dept_manager→departments |         0
 salaries→employees       |         0
 titles→employees         |         0
(6 filas)
```

### 3.2 Comparación de contenido entre motores

No basta con que los conteos coincidan, por lo que se compararon agregados sobre los datos en ambos motores:

```sql
SELECT COUNT(*), SUM(salary), MIN(from_date), MAX(to_date) FROM salaries;
SELECT gender, COUNT(*), MIN(birth_date), MAX(hire_date) FROM employees GROUP BY gender ORDER BY gender;
```

| Prueba | MariaDB | PostgreSQL 18 | Estado |
| --- | --- | --- | --- |
| `salaries`: filas / suma / mín. `from_date` / máx. `to_date` | 2844047 / 181480757419 / 1985-01-01 / 9999-01-01 | 2844047 / 181480757419 / 1985-01-01 / 9999-01-01 | Coincide |
| `employees` género M: filas / mín. nacimiento / máx. contratación | 179973 / 1952-02-01 / 2000-01-28 | 179973 / 1952-02-01 / 2000-01-28 | Coincide |
| `employees` género F: filas / mín. nacimiento / máx. contratación | 120051 / 1952-02-02 / 2000-01-22 | 120051 / 1952-02-02 / 2000-01-22 | Coincide |

### 3.3 Análisis de resultados

Los conteos de las seis tablas coinciden exactamente entre MariaDB y PostgreSQL, con un total de 3,919,015 registros. La comparación de contenido (suma de salarios, fechas mínimas y máximas, y distribución por género) también coincide, lo que indica que los datos se transfirieron sin alteraciones y no solo en cantidad. Las seis consultas de huérfanos devolvieron 0, y las seis llaves foráneas se crearon sin errores, por lo que PostgreSQL validó las relaciones sobre los datos existentes. Las dos vistas migradas devuelven el mismo número de filas (300,024) y la misma muestra ordenada que en MariaDB.

Se encontró una discrepancia durante la verificación: las tablas `employees` y `departments` habían quedado vacías en PostgreSQL y la estructura no tenía llaves primarias ni foráneas. Se resolvió recargando únicamente esas dos tablas en modo solo datos y agregando después las llaves primarias y foráneas. Esto muestra la importancia de verificar los conteos tabla por tabla y no dar por buena una migración solo porque el proceso termine sin errores.

---

## 4. Respaldos y Repositorio

### 4.1 Respaldos

```bash
pg_dump -h 127.0.0.1 -p 5433 -U juan -d pdb_employees -F c -f pdb_employees_backup.dump
mysqldump -h 127.0.0.1 -u root -p employees > employees_backup_mariadb.sql
ls -lsh pdb_employees_backup.dump employees_backup_mariadb.sql
```

```text
165M -rw-r--r-- 1 jgabriel jgabriel 165M sep 28 19:17 employees_backup_mariadb.sql
 34M -rw-r--r-- 1 jgabriel jgabriel  34M sep 28 19:17 pdb_employees_backup.dump
```

Ambos respaldos se generaron después de corregir la base, por lo que el de PostgreSQL incluye los datos completos, las llaves primarias y foráneas, y las tres vistas.

### 4.2 Repositorio

Los archivos de respaldo (`*.dump`, `*.sql`) se excluyeron del repositorio mediante `.gitignore` por su tamaño, y las contraseñas se reemplazaron por `TU_PASSWORD` en todos los archivos publicados.

---

## 5. Eje Transversal y ODS

* **Tecnologías Emergentes y Adaptabilidad Digital:** Implementación de flujos de trabajo sobre plataformas contenedorizadas en Docker, facilitando entornos portables e interoperables mediante herramientas de línea de comandos.
* **ODS 4 (Educación de Calidad):** Desarrollo de competencias analíticas y técnicas aplicadas a la gestión de bases de datos.
* **ODS 10 (Reducción de Desigualdades):** Adopción de software libre y tecnologías Open Source (PostgreSQL, MariaDB, Docker) para democratizar la infraestructura técnica.

---

## 6. Conclusiones

1. `pgloader` resultó adecuado para transferir de forma automatizada cerca de 3.9 millones de registros, pero su reporte de finalización no sustituye la verificación: fue la comparación de conteos entre ambos motores la que reveló que dos tablas habían quedado vacías.
2. La estructura destino no incluía llaves primarias ni foráneas. Crear primero las primarias y luego las foráneas, con los datos ya cargados, permitió además validar la integridad referencial, ya que PostgreSQL rechaza una llave foránea si existen huérfanos.
3. La migración de vistas requirió adaptar la sintaxis de MariaDB (backticks, `DEFINER`, `ALGORITHM`, `SQL SECURITY`) y respetar el orden de dependencia: `current_dept_emp` se apoya en `dept_emp_latest_date`.
4. La validación combinó conteos, agregados de contenido, búsqueda de huérfanos y comparación de vistas en ambos motores, lo que da mayor confianza que una sola prueba de volumen.

---

## 7. Referencias (Normas APA 7)

* MariaDB Foundation. (2026). *MariaDB Knowledge Base: SHOW CREATE VIEW*. https://mariadb.com/kb/en/show-create-view/
* PostgreSQL Global Development Group. (2026). *PostgreSQL 18 Documentation: SQL Commands and Views*. https://www.postgresql.org/docs/18/
