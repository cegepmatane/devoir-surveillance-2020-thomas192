# Devoir Surveillance

Thomas Saudemont

## Procédures stockées

### Journalisation

#### Table 'chaindemontagne'

````sql
CREATE OR REPLACE FUNCTION journaliser() 
	RETURNS trigger
    LANGUAGE plpgsql
AS $$
DECLARE 
    operation text;
	description text;
    objetAvant text;
    objetApres text;
BEGIN
	objetAvant := '';
	objetApres := '';

	IF TG_OP = 'INSERT' THEN
    	objetAvant := '{}';
   		objetApres := '{'||NEW.id||','||NEW.nom||','||NEW.description||'}';
        operation := 'AJOUTER';
    END IF;
	IF TG_OP = 'DELETE' THEN
    	objetAvant := '{'||OLD.id||','||OLD.nom||','||OLD.description||'}';
    	objetApres := '{}';
        operation := 'SUPPRIMER';
    END IF;
    IF TG_OP = 'UPDATE' THEN
    	objetAvant := '{'||OLD.id||','||OLD.nom||','||OLD.description||'}';
   		objetApres := '{'||NEW.id||','||NEW.nom||','||NEW.description||'}';
        operation := 'MODIFIER';
    END IF;

	description := objetAvant || ' -> ' || objetApres;
	INSERT into journal(moment, objet, operation, description) VALUES(NOW(), 		 'chainedemontagne', operation, description);
    
	IF TG_OP = 'DELETE' THEN
		return OLD;
	END IF; 
    return NEW;
END
$$;
````

````sql
CREATE TRIGGER journaliserAjout BEFORE INSERT ON chainedemontagne FOR EACH ROW EXECUTE PROCEDURE journaliser();
CREATE TRIGGER journaliserSuppression BEFORE DELETE ON chainedemontagne FOR EACH ROW EXECUTE PROCEDURE journaliser();
CREATE TRIGGER journaliserModification BEFORE UPDATE ON chainedemontagne FOR EACH ROW EXECUTE PROCEDURE journaliser();
````

### Statistiques

````sql
CREATE OR REPLACE PROCEDURE statistiques()
LANGUAGE plpgsql
AS $$
DECLARE
	col1 bigint;
	col2 bigint;
	col3 bigint;
	col4 bigint;
	checksum text;
	nombre integer;
	altitudeMoyenne integer;
BEGIN
/* ---------------------------------- TABLE SOMMET ---------------------------------- */
    
    /* Checksum */
    SELECT SUM(('x' || substring(hash_1, 1, 8))::bit(32)::bigint) INTO col1
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("altitude"::text), '') ||
        coalesce(md5("id_chainedemontagne"::text), '')
        ) AS hash_1
        FROM sommet
    ) AS hash_2;

    SELECT SUM(('x' || substring(hash_1, 9, 8))::bit(32)::bigint) INTO col2
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("altitude"::text), '') ||
        coalesce(md5("id_chainedemontagne"::text), '')
        ) AS hash_1
        FROM sommet
    ) AS hash_2;

    SELECT SUM(('x' || substring(hash_1, 17, 8))::bit(32)::bigint) INTO col3
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("altitude"::text), '') ||
        coalesce(md5("id_chainedemontagne"::text), '')
        ) AS hash_1
        FROM sommet
    ) AS hash_2;

    SELECT SUM(('x' || substring(hash_1, 25, 8))::bit(32)::bigint) INTO col4
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("altitude"::text), '') ||
        coalesce(md5("id_chainedemontagne"::text), '')
        ) AS hash_1
        FROM sommet
    ) AS hash_2;

    checksum := col1 + col2 + col3 + col4;
    checksum := md5(checksum);
	
	/* Statistiques */
    SELECT COUNT(id) INTO nombre FROM sommet;
    SELECT AVG(altitude) INTO altitudeMoyenne FROM sommet;

    /* Insertion */
    INSERT INTO statistiques_sommet(moment, nombre, altitudemoyenne, checksum)
    VALUES(NOW(), nombre, altitudeMoyenne, checksum);
    
/* ----------------------------- TABLE CHAINEDEMONTAGNE ----------------------------- */
	/* Checksum */
    SELECT SUM(('x' || substring(hash_1, 1, 8))::bit(32)::bigint) INTO col1
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("description"::text), '')
        ) AS hash_1
        FROM chainedemontagne
    ) AS hash_2;

    SELECT SUM(('x' || substring(hash_1, 9, 8))::bit(32)::bigint) INTO col2
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("description"::text), '')
        ) AS hash_1
        FROM chainedemontagne
    ) AS hash_2;

    SELECT SUM(('x' || substring(hash_1, 17, 8))::bit(32)::bigint) INTO col3
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("description"::text), '')
        ) AS hash_1
        FROM chainedemontagne
    ) AS hash_2;

    SELECT SUM(('x' || substring(hash_1, 25, 8))::bit(32)::bigint) INTO col4
    FROM (
        SELECT md5(
        coalesce(md5("id"::text), '') ||
        coalesce(md5("nom"::text), '') ||
        coalesce(md5("description"::text), '')
        ) AS hash_1
        FROM chainedemontagne
    ) AS hash_2;

    checksum := col1 + col2 + col3 + col4;
    checksum := md5(checksum);
	
	/* Statistiques */
    SELECT COUNT(id) INTO nombre FROM chainedemontagne;

    /* Insertion */
    INSERT INTO statistiques_chainedemontagne(moment, nombre, checksum)
    VALUES(NOW(), nombre, checksum);
END; 
$$
````

## Plan de contingence

https://docs.google.com/presentation/d/1mf8oiCBA9EW3z1SLxw08y0fmODLPxwxNhG86KISmp3c/edit?usp=sharing

### Sauvegarde des données

#### Automatisation de la sauvegarde

Sauvegarde de la base de données 'neo' toutes les minutes

```
1-59 * * * * pg_dump --no-password -U postgres neo > /home/thomas/backups/backup_$(date 	+\%Y\%m\%d\%H\%M\%S).pgsql
```

#### Restauration

https://youtu.be/X62yIiTkbeM

```
createdb -U postgres -T template0 neobis
psql -U postgres neobis < /home/thomas/db_backups/backup_20201117162501.txt.pgsql
```

