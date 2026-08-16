# 1 Validation

- [RAP Dokumentation - Validation](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_VALIDATIONS.html)



## 1.1 Validation Email
**Szenario:** Wir benötigen eine Validierung, die nur retailsolutions E-Mail Adressen zulässt bei einem Mitarbeiter.

In der ZI_RAP_XX_EMPLOYEE behavior Definition ergänzen wir bei 'draft determine action Prepare' folgendes:
<figure style="border:1px solid #aaa; padding:10px; display:inline-block;">
  <img src="pictures/validation_1.png" 
       alt="Markdown Logo" min-width:150px; width: 40%>
</figure>


Über die Glühlampe können wir uns die Methode generieren lassen, oder wir wechseln in die ZBP_I_RAP_XX_EMPLOYEE und legen sie selbst an.
<figure style="border:1px solid #aaa; padding:10px; display:inline-block;">
  <img src="pictures/validation_2.png" 
       alt="Markdown Logo" min-width:150px; width: 40%>
</figure>

Wenn wir jetzt versuchen eine Email mit einer anderen Endung anzugeben, bekommen wir einen Fehler beim Speichern.

<figure style="border:1px solid #aaa; padding:10px; display:inline-block;">
  <img src="pictures/validation_3.png" 
       alt="Markdown Logo" min-width:150px; width: 40%>
</figure>

>**HINWEIS** Eclipse hat auch einen Debugger. Gerne mal einen Breakpoint in Zeile mit der "If NOT" Bedingung der checkMail Methode setzen und ihn aus dem Frontend triggern (Trigger durch eine Falsche Email und dann Klick auf Save).

### 1.2 EML 
- [EML - Read Entities](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABAPREAD_ENTITIES_LONG.html)

EML (Entity Manipulation Language) ist eine auf ABAP basierende und speziell für RAP entwickelte Programmiersprache. Mit EML können ABAP-Programme type-sicher auf RAP-Geschäftsobjekte zugreifen und deren Daten manipulieren.

Vorteile:
- Einhaltung der Business-Logik: Anders als beim direkten Lesen (z.B. über klassisches Open SQL) werden bei EML alle Validierungen, Berechnungen (Determinations) und Einstellungen durchlaufen, die im RAP-Verhaltensmodell (Behavior Definition) definiert sind. Sie umgehen Ihre eigene Geschäftslogik also nicht.
- Draft-Unterstützung: EML ermöglicht es, Daten aus dem Draft-Puffer (nicht gespeicherte Änderungen in Fiori-Apps) und der echten Datenbank zu lesen und zu bearbeiten.
- Ersatz für BAPIs: Es bietet eine standardisierte Schnittstelle, um alte ABAP-Programme oder Klassen nahtlos an moderne RAP-Geschäftsobjekte anzubinden.

### 1.3 Validation checkEntryDate
Wir überprüfen, ob das Eintrittsdatum in der Vergangenheit liegt.

Dazu fügen wir wieder in der ZI_RAP_XX_EMPLOYEE Behavior Definition folgendes hinzu:

<figure style="border:1px solid #aaa; padding:10px; display:inline-block;">
  <img src="pictures/validation_5.png" 
       alt="Markdown Logo" min-width:150px; width: 40%>
</figure>



Oben wieder die Methode hinzufügen:
>IMPORTING keys FOR Employee~checkMail. METHODS checkEntryDate FOR VALIDATE ON SAVE IMPORTING keys FOR Employee~checkEntryDate.

Sowie unten die Methode:
<figure style="border:1px solid #aaa; padding:10px; display:inline-block;">
  <img src="pictures/validation_4.png" 
       alt="Markdown Logo" min-width:150px; width: 40%>
</figure>

# 2. Determinations 
[RAP Dokumentation - Determination](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_DETERMINATIONS.html)

## 2.1 Calcnames
**Szenario:** Bei der Auswahl des Usernames soll automatisch der Vorname & Nachname belegt werden.
Das UI soll sich live updaten im Draft Modus.

**MODIFY**
```eml
MODIFY ENTITIES OF bdef (IN LOCAL MODE)
ENTITY entityName
UPDATE FIELDS (field1, field2) WITH update_tab_structure
```

**Generierte Datentypen vom Framework - Bdef Derived Types**
[TYPE TABLE FOR](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABAPTYPE_TABLE_FOR.html)
```abap
DATA: lt_update_employees TYPE TABLE FOR UPDATE zi_rap_XX_employee.
```

**Problem: Updates kommen nicht im UI an**

#### Side Effects 🌊
[Side effects](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_SIDE_EFFECTS.html)

Mit Hilfe von Side Effects können wir dem Framework mitteilen, dass es gewisse Entitäten, Felder, Nachrichten im UI nachladen muss.

**Syntax**
```bdef
side effects {
  field MyField affects Targets;
  ...
}
```

Folgende Targets können benutzt werden:
- Felder,
- Berechtigungen,
- die auslösende Entität $self
- eine andere Entität
- Nachrichten

Zusätzlich müssen Side Effects in der Consumption View aktiviert werden
```bproj
use side effects;
```
#### Stolperfallen 🚧
**Problem: Determination wird bei Feldänderung im Draft und bei Speichern ausgeführt - Nutzereingaben werden eventuell wieder überschrieben**

 IF keys[ 1 ]-%is_draft = if_abap_behv=>mk-off.
      RETURN.
    ENDIF.

    READ ENTITIES OF zi_rap_xx_employee IN LOCAL MODE
    ENTITY Employee
    ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(lt_employee)
    FAILED DATA(lt_employee_failed).

    IF lines( lt_employee ) < 1.
        RETURN.
    ENDIF.

    DATA(ls_employee) = lt_employee[ 1 ].

    DATA: lt_employee_update TYPE TABLE FOR UPDATE zi_rap_xx_employee.

    SELECT SINGLE * FROM ZI_RAP_WS_USER_VH
    WHERE bname = @ls_employee-username INTO @DATA(ls_user).

    IF ls_user-NameFirst IS NOT INITIAL OR ls_user-NameLast IS NOT INITIAL.
      lt_employee_update = CORRESPONDING #( lt_employee ).
      LOOP AT lt_employee_update ASSIGNING FIELD-SYMBOL(<ls_update>).
        <ls_update>-Firstname = ls_user-NameFirst.
        <ls_update>-Surname = ls_user-NameLast.
      ENDLOOP.

      MODIFY ENTITIES OF zi_rap_xx_employee IN LOCAL MODE
      ENTITY Employee
      UPDATE FIELDS ( Firstname Surname ) WITH lt_employee_update
      FAILED DATA(lt_update_failed)
      REPORTED DATA(lt_updated_reported).

    ENDIF.

## 2.2 ProposeWorkload
🎬 **Szenario:** Wenn ein Projekt angelegt wird soll die Auslastung in dem Projekt vorbelegt werden. Dabei gehen wir von einer Standardauslastung von 5 Tagen aus. Ist der Nutzer bereits in zwei Projekten mit jeweils 2 Tagen gebucht, soll die Determination bei Auswahl eines Projektes die Auslastung mit einem Tag vorbelegen (5-2-2 = 1).

**Hinweis**
Lese das neu angelegte Projekt und die Root-Entität Employee.
Mit Hilfe der Keys der Root-Entität lassen sich die anderen Projekte laden

**BDef EmplProjects**
```bdef
  determination proposeWorkload on modify { field ProjectId; }
```

**Methode proposeWorkload**
```abap
    DATA: lv_proposed_workload TYPE z_de_rap_ws_days_per_week.

    IF keys[ 1 ]-%is_draft = if_abap_behv=>mk-off.
      RETURN.
    ENDIF.

    READ ENTITIES OF zi_rap_xx_employee
    IN LOCAL MODE
    ENTITY EmplProjects
    ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(lt_edit_project)
    BY \_Employee
    ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(lt_employee).

    IF lt_employee IS INITIAL OR lines( lt_edit_project ) < 1.
      RETURN.
    ENDIF.

    DATA(lv_user_id) = lt_employee[ 1 ]-%tky-UserUuid.

    READ ENTITIES OF zi_rap_xx_employee
    IN LOCAL MODE
    ENTITY Employee
    BY \_EmpProjects
    ALL FIELDS WITH CORRESPONDING #( lt_employee )
    RESULT DATA(lt_projects).

    DATA(lv_workload) = '0.0'.

    LOOP AT lt_projects ASSIGNING FIELD-SYMBOL(<ls_empl_project>).
      lv_workload += <ls_empl_project>-Workload.
    ENDLOOP.

    IF lv_workload < '5.0'.
      lv_proposed_workload = '5.0' - lv_workload.
    ENDIF.

    DATA: lt_update_project TYPE TABLE FOR UPDATE zi_rap_xx_proj_emp.

    lt_update_project = CORRESPONDING #( lt_edit_project ).
    lt_update_project[ 1 ]-Workload = lv_proposed_workload.

    MODIFY ENTITIES OF zi_rap_xx_employee IN LOCAL MODE
        ENTITY EmplProjects
        UPDATE
        FIELDS ( Workload ) WITH lt_update_project
        FAILED DATA(lt_project_failed)
        REPORTED DATA(lt_project_reported).
```

```bdef
 side effects { field ProjectId affects field Workload; }
```

# 3. Feldsteuerung und Action
## 3.1 Feldsteuerung (Feature Instance)
- [RAP Dokumentation - Field Characteristics](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_FIELD_CHAR.html)

- Auch für Actions, Draft Actions und Standard-Buttons möglich
- Implementierung erfolgt innerhalb des Behavior Pools der jeweiligen Entität in der Methode `get_instance_features` (kann automatisch generiert werden)

**Syntax innerhalb der Behavior Definition**
```bdef
  field ( features : instance ) <Liste mit Feldnamen>
  action ( features : instance ) <Definition der Action>;
```

### 3.1.1 Felder 
- <b>🎬Szenario:</b> Wenn Name und Vorname im SAP Nutzer schon eingetragen sind, dann sind die Felder vom Nutzer nicht mehr pflegbar

### Verhaltensdefinition
<b>Felder definieren</b>
```bdef
field ( features : instance ) Firstname, Surname;
```

### Verhaltensimplmentierung
- Implementierung in `get_instance_features` (kann generiert werden)

    **EML READ**
    ```abap
    READ ENTITIES OF zi_rap_xx_employee IN LOCAL MODE
        ENTITY Employee
        FIELDS ( Username Firstname Surname )
        WITH CORRESPONDING #( keys )
        RESULT DATA(lt_employee)
        FAILED DATA(lt_employee_failed).

    ```

     **Setze Vor- und Nachname readonly**
    ```abap
    ls_req_fields-Firstname = if_abap_behv=>fc-f-read_only.
    ls_req_fields-Surname = if_abap_behv=>fc-f-read_only.
    ```

    **Befüllen der result Strutktur**
    ```abap
    INSERT VALUE #(
          %tky = <ls_employee>-%tky " setzt die Verbindung zur Instanz
          %field = ls_req_fields
        ) INTO TABLE result.
    ```

### Exkurs ABAP Cloud
- Lesen der Nutzer Daten über den Custom CDS View, kurzer Exkurs warum Custom CDS View

    <b>Beispiel für klassischen SELECT zeigen, der syntaktisch hier nicht geht</b>
    ```abap
    SELECT SINGLE * FROM usr21 AS user
        JOIN adrp AS adrp ON user~persnumber = adrp~persnumber
        WHERE bname = @<ls_employee>-Username
        INTO @DATA(ls_user).
    ```

    <b>SELECT auf dem neuen Custom CDS View </b>
    ```abap
          SELECT SINGLE *
          FROM ZI_RAP_WS_USER_VH
          WHERE Bname = @<ls_employee>-Username
          INTO @DATA(ls_user).
    ```

**❗Problem❗**
Der Side Effect den wir vorher auf den Feldern für Vorname und Nachname definiert hatten, übernimmt noch nicht die readonly Information. Es muss nochmal ein read getriggert werden.

**🖊️Aufgabe🖊️**
Passe den Side Effect so an, dass er auch nach der Action triggert.

**ℹ️Hinweisℹ️**
Es genügt die Anpassung in der Verhaltensdefinition. Schau in die Doku 👀📖


### 3.1.2 Aufgabe Standard Buttons
- <b>🎬Szenario:</b> Nur der Nutzer selbst kann seine Nutzerdaten editieren und seinen Nutzer Eintrag löschen. Dazu müssen die Standard Buttons "Bearbeiten" und "Löschen" in `get_feature_instance` aufgenommen werden.
- <b>ℹ️Hinweis:</b> Der "Löschen"-Button hängt an der Standard `delete`-Operation und der "Bearbeiten"-Buttton an der Draft Action `Edit`. Das wirkt sich auch auf die Strukturen `requested_features` und `result` aus.

**✅Checkliste**
- Actions in Verhaltensdefinition für `requested_features` bekannt machen
- Logik implmentieren
- Eintrag in `result` ergänzen


## 3.2. Teil: Actions
- 📖[RAP Dokumentation - Action](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_ACTION1_ABEXA.html)
- 📖[RAP Dokumentation - Action mit Parametern](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_ACTION2_ABEXA.html)
- 📖[RAP Dokumentation - action (Syntax)](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENBDL_ACTION.html)


### 3.2.1 Teil: Action "Resign"
- <b>🎬Szenario:</b> Es gibt auf der Detail Page einen Button mit der Mitarbeiter kündigen kann. Dieser setzt das Austrittsdatum auf Tagesdatum plus sechs Monate.

### Verhaltensdefinition

**Definition im Interface View**
```bdef
action resign result [1] $self;
```
**Bekanntmachen im Consumption View**
```bdef
use action resign;
```

### Metadatenerweiterung

**Einbinden des Buttons per Annotation**
```cds
@UI.identification: [{
    position: 10,
    type : #FOR_ACTION,
    dataAction: 'resign',
    label : 'Kündigen!'
}]
```

### Implementierung - Exkurs: ABAP Cloud

**Ermittlung des Datums ohne sy-Struktur**
```abap
DATA(lv_date_today) = cl_abap_context_info=>get_system_date( ).
```

**Beispiel klassischer Fuba**
```abap
DATA lv_new_date  LIKE sy-datum.
CALL FUNCTION 'RE_ADD_MONTH_TO_DATE'
    EXPORTING
    months  = '6'
    olddate = sy-datum
    IMPORTING
    newdate = lv_new_date.
```

### Implementierung - EML Statement

**EML Update mit inline FOR**
```abap
MODIFY ENTITIES OF zi_rap25_employee IN LOCAL MODE
    ENTITY Employee
    UPDATE FIELDS ( ExitDate )
    WITH VALUE #( FOR key IN keys ( %tky = key-%tky ExitDate = lv_new_date ) ).
```

### Implementierung - APPEND der RESULT Struktur

**APPEND in die RESULT Struktur**
```abap
LOOP AT keys ASSIGNING FIELD-SYMBOL(<ls_key>).
    INSERT VALUE #(
        %tky = <ls_key>-%tky
        %param-UserID = <ls_key>-UserID
    ) INTO TABLE result.
ENDLOOP.
```

### 3.2.2 Übung: Action "quitProject"
- <b>🎬Szenario:</b> Der Mitarbeiter kann in seiner Liste mit Projekten eines oder mehrere auswählen und diese zu einem bestimmten Datum verlassen. Dazu hat die Action noch einen Parameter um das Datum anzugeben.
- <b>ℹ️Hinweise:</b> 
    - Die Action muss dann auf der Projekt Entität definiert und implmentiert werden. 
    - Parameter müssen als `Abstract Entities` im CDS definiert werden.
    - Parameter findet ihr in der `keys`-Tabelle
    - In EML Statements kann FOR...IN Syntax verwendet werden (Alternativ zur Tabelle mit Update Struktur wie im Derterminations Teil vorgestellt)
    - In der Metadatenerweiterung muss die Action in `UI.LineItem` eingebunden werden

    **Abstract Entity für Parameter**
    ```cds
    @EndUserText.label: 'Abstract Entity for Quitting Date'
    define abstract entity ZI_RAP_XX_QUIT_DATE
    {
        @EndUserText.label: 'Austrittsdatum'
        quit_date : abap.dats;
    }
    ```

**✅Checkliste**
- `abstract entity` für die Parameter anlegen 
- Action in der Verhaltensdefinition definieren (Syntax siehe Doku)
- Methode im Behavior Pool anlegen lassen
- Metadatenerweiterung nutzen um Button ins UI einzubinden
- Logik implmentieren (Parameter auslesen und EML-Update Statement)
- `result` nicht vergessen

### 3.3 Übung: Feature Instances für die Actions
- <b>🎬Szenario:</b>: Die implmentierten Action sollen noch in die Feldsteuerung mit aufgenommen werden. Kündigen soll nur im Anzeige-Modus verfügbar sein und jeder Nutzer soll nur für sich selbst kündigen können.
- <b>ℹ️Hinweise:</b> 
    - Die Information ob man sich im Draft befindet oder nicht ist in der `keys`-Tabelle oder im Resultat des READ ENTITIES vorhanden. 
    - Beim Prüfen die korrekte Wahl der Struktur aus `if_abap_behav` beachten.

**✅Checkliste:**
- Actions in Verhaltensdefinition für `requested_features` bekannt machen
- Logik implmentieren
- Eintrag in `result` ergänzen

### 3.4 Übung Action "Resign" erweitern (optional)
- <b>🎬Szenario:</b> Bei einer Kündigung müssen natürlich auch die Projekte zum Austrittsdatum verlassen werden. Die Aktion "Resign" muss dementsprechend erweitert werden

