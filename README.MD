# Devoir Surveillance

## Procédures stockées

### Journalisation

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
	INSERT into journal(moment, objet, operation, description) VALUES(NOW(), 'chainedemontagne', operation, description);
    
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

