
# T03: Servidor de Fitxers – Informe Tècnic

**Informe Tècnic – T03: Servidor de Fitxers**  
**Domini:** FoodLogistic.test  
**Nom del servidor:** FL07  
**Assignatura:** Sistemes Operatius en Xarxa  
**Nom Alumne:** Lluís García Martínez

---

## 1. Estructura d'Unitats Organitzatives i Grups de Seguretat (Active Directory)

### 1.1 Objectiu

Preparar l'Active Directory amb una estructura lògica i escalable abans de desplegar el servidor de fitxers, seguint el principi de **menor privilegi** (*least privilege*) i facilitant l'aplicació de GPOs per departament.

### 1.2 Raonament

Una estructura adequada d'Unitats Organitzatives (OUs) permet les següents accions:

- Aplicar **GPOs** de forma granular, amb la possibilitat de configurar-ne una per departament.
- **Delegar** l'administració d'usuaris als caps de departament sense atorgar permisos de domini.
- Mantenir l'ordre i la traçabilitat en entorns que experimenten creixement.

### 1.3 Estructura d'OUs implementada

```
foodlogistic.test
├── OU=FoodLogistic
│   ├── OU=Grups
│   │   ├── Group=Administracio
│   │   ├── Group=Transport
│   │   └── Group=Direccio
│   ├── OU=Usuaris
```

![alt text](<picsf2/Captura de pantalla 2026-04-14 163138.png>)

### 1.4 Grups de Seguretat creats (Global Security Groups)

| Nom del grup | Tipus | Ubicació | Finalitat |
|-------------|-------|----------|-----------|
| `Administracio` | Global Security | OU=Grups | Gestió de factures i albarans |
| `Transport` | Global Security | OU=Grups | Xofers i caps de flota |
| `Direccio` | Global Security | OU=Grups | Gerència i accés a documentació confidencial |

Aquests grups són **Global Security Groups** i s'utilitzen tant per assignar permisos NTFS/SMB com per filtrar l'aplicació de GPOs.

---

## 2. Resum de Configuració

| Carpeta | Camí físic | Camí UNC | Grups amb accés | Mètode de creació | Permisos efectius (xarxa) | Observacions |
|---------|-----------|----------|----------------|-------------------|---------------------------|--------------|
| **Public** | `C:\Public` | `\\FL07\Public` | Tothom (Everyone) | Explorador de fitxers | **Lectura** | Sense ABE |
| **Operacions** | `C:\Operacions` | `\\FL07\Operacions` | `Transport` | Server Manager (FSSM) | **Modificació** + ABE | Només visible per Transport |
| **Direccio** | `C:\Direccio` | `\\FL07\Direccio` | `Direccio` | **PowerShell Avançat (Opció D)** | **Control Total** + ABE | Unitat **C:** via GPO |

**Servidor:** FL07.foodlogistic.test (Windows Server 2022)

![alt text](<picsf2/Captura de pantalla 2026-04-14 163256.png>)

---

## 3. Implementació de Recursos Compartits

### 3.1 Carpeta Public (Mètode: Explorador de fitxers)

#### 3.1.1 Objectiu

Crear una carpeta accessible per a tots els usuaris de l'empresa utilitzant **exclusivament** l'Explorador de Windows, tal com exigeix l'enunciat.

#### 3.1.2 Raonament

Aquest apartat demostra el domini de la via gràfica bàsica. L'enunciat proposa una combinació intencionada de permisos SMB i NTFS perquè es pugui observar com es calculen els **permisos efectius**.

#### 3.1.3 Procediment detallat

**Pas 1: Creació de la carpeta física**

- Connectar-se al servidor `FL07` amb un compte d'**Administrador de Domini**.
- Obrir l'**Explorador de fitxers** i navegar a la unitat `C:\`.
- Clic dret sobre `C:\` > **Nou** > **Carpeta** > Assignar el nom **`Public`**.
- Ruta resultant: `C:\Public`

**Pas 2: Configuració dels permisos de compartició (SMB)**

- Clic dret sobre `C:\Public` > **Propietats** > Pestanya **Compartir** > **Compartir...**
- A la llista desplegable, seleccionar **Everyone** i assignar el nivell **Lectura**.
- Clic a **Compartir** i, posteriorment, **Fet**.

![alt text](<picsf2/Captura de pantalla 2026-05-22 173742.png>)

**Pas 3: Configuració dels permisos NTFS**

- A la mateixa finestra de **Propietats**, accedir a la pestanya **Seguretat**.
- Clic a **Edita** > **Afegir**.
- Escriure `Everyone` i assignar el permís **Modificació** (Modify).
- Assegurar que el grup **Administradors** disposi de **Control total**.
- Clic a **Aplica** i **D'acord**.

![alt text](<picsf2/Captura de pantalla 2026-05-22 173934.png>)

#### 3.1.4 Anàlisi tècnica: Càlcul del permís efectiu

Quan un usuari accedeix a una carpeta compartida per xarxa, Windows aplica **dues capes de seguretat**:

1. **Permisos SMB** (de la compartició).
2. **Permisos NTFS** (del sistema de fitxers).

La regla aplicable és la següent: el permís efectiu correspon a **el més restrictiu** de tots dos.

| Capa | Permís assignat |
|------|----------------|
| SMB (Share) | Everyone > **Lectura** |
| NTFS | Everyone > **Modificació** |
| **Permís efectiu final** | **Lectura** |

Tot i que NTFS permetria escriure, la capa SMB restringeix l'accés a només lectura. En entorns professionals, és habitual configurar **Canvi (Change)** a SMB i detallar els permisos a nivell NTFS.

---

### 3.2 Carpeta Operacions (Mètode: Server Manager – FSSM)

#### 3.2.1 Objectiu

Crear el recurs compartit `Operacions` des de la consola **Server Manager**, activant **Access-Based Enumeration (ABE)** i restringint l'accés exclusivament al grup `Transport`.

#### 3.2.2 Raonament

El Server Manager ofereix una interfície més professional i completa que l'Explorador. Permet crear el share i la carpeta simultàniament, i configurar ABE directament durant el procés.

#### 3.2.3 Procediment detallat

1. Obrir **Server Manager** > **File and Storage Services** > **Shares**.
2. Al panell de la dreta, clic a **TASKS** > **New Share...**
3. Seleccionar el perfil **SMB Share – Quick**.
4. A **Share location**, triar el servidor i especificar la ruta personalitzada:
   ```
   C:\Operacions
   ```
   Si la carpeta no existeix, l'assistent la crea automàticament.

5. A **Share name**, introduir: `Operacions`.
6. A la pantalla **Other settings**, marcar la casella següent:
   - **Enable access-based enumeration**
   
   Això farà que els usuaris només vegin aquesta carpeta si tenen permisos explícits sobre ella.

![alt text](<picsf2/Captura de pantalla 2026-05-22 174659.png>)

![alt text](<picsf2/Captura de pantalla 2026-05-22 174620.png>)

7. A la pantalla **Permissions**, eliminar els permisos per defecte i afegir **només** el grup `Transport` amb permisos de **Modificació** (Modify).
   - Assegurar que **Administradors** mantingui **Control total**.

![alt text](<picsf2/Captura de pantalla 2026-05-22 174948.png>)

8. Revisar el resum i clicar **Create**.

#### 3.2.4 Anàlisi tècnica

L'**Access-Based Enumeration (ABE)** és una funcionalitat del protocol SMB que filtra la llista de fitxers i carpetes abans d'enviar-la al client. Si l'usuari no té permís de lectura sobre un element, aquest no apareix en l'explorador de xarxa. Això millora la seguretat i evita confusions.

---

### 3.3 Carpeta Direccio (Mètode: PowerShell Avançat – Opció D)

#### 3.3.1 Objectiu

Crear una carpeta confidencial només accessible pel grup `Direccio`, utilitzant **PowerShell** amb ABE, i configurar una unitat de xarxa automàtica mitjançant **GPO**.

#### 3.3.2 Raonament

Aquest és el mètode més avançat i professional. Permet automatitzar la creació, aplicar configuracions complexes (com ABE via cmdlet) i deixa tot documentat en script per si cal replicar l'entorn.

#### 3.3.3 Procediment detallat i codi

**Pas 1: Crear la carpeta física**

```powershell
New-Item -Path "C:\Direccio" -ItemType Directory -Force
```

**Pas 2: Crear el recurs compartit amb ABE activat**

El valor correcte per a l'enumeració basada en accés és `AccessBased` (amb doble 's').

```powershell
New-SmbShare -Name "Direccio" `
             -Path "C:\Direccio" `
             -FullAccess "Direccio" `
             -FolderEnumerationMode AccessBased `
             -CachingMode None `
             -Description "Carpeta confidencial solo para Direccion"
```

![alt text](<picsf2/Captura de pantalla 2026-06-01 201555.png>)

**Pas 3: Reforçar permisos NTFS**

```powershell
# Eliminar herència i netejar permisos heretats
icacls "C:\Direccio" /inheritance:r

# Assignar Control Total al grup Direccio (OI=Object Inherit, CI=Container Inherit)
icacls "C:\Direccio" /grant "Direccio:(OI)(CI)F"

# Assignar Control Total als Administradors
icacls "C:\Direccio" /grant "Administradores:(OI)(CI)F"
```

![alt text](<picsf2/Captura de pantalla 2026-06-01 201746.png>)

**Pas 4: Verificació del share creat**

```powershell
Get-SmbShare -Name "Direccio" | Format-List
```

Resultat esperat: `FolderEnumerationMode : AccessBased`

#### 3.3.4 Configuració de la GPO per mapar la unitat C:

1. Obrir **Group Policy Management** (`gpmc.msc`).
2. Crear una nova GPO anomenada: `Map Drive Direccio - C:`.
3. Enllaçar-la a l'OU `Usuaris\Direccio`.
4. Al filtre de seguretat, eliminar **Authenticated Users** i afegir **només** el grup `Direccio`.
5. Editar la GPO i navegar a:
   ```
   User Configuration > Preferences > Windows Settings > Drive Maps
   ```
6. Clic dret > **New** > **Mapped Drive**:
   - **Action:** Create
   - **Location:** `\\FL07\Direccio`
   - **Reconnect:** Enabled
   - **Label as:** Direccio
   - **Drive Letter:** C:
   - **Hide/Show this drive:** Show this drive

![alt text](<picsf2/Captura de pantalla 2026-06-01 202836.png>)

7. Tancar l'editor i forçar l'actualització de la política als clients:
   ```cmd
   gpupdate /force
   ```


---

## 4. Control d'Emmagatzematge (Quotas i FSRM)

### 4.1 Quota NTFS (per volum – C:)

#### 4.1.1 Objectiu

Limitar l'espai que pot ocupar cada usuari nou al volum de dades.

#### 4.1.2 Procediment

1. Obrir l'**Explorador de fitxers** > Clic dret sobre la unitat `C:` > **Propietats**.
2. Pestanya **Quota** > Clic a **Show Quota Settings**.
3. Activar **Enable quota management**.
4. Marcar **Deny disk space to users exceeding quota limit**.
5. Seleccionar **Limit disk space to**: `500 MB`.
6. Establir el llindar d'avís a `85%`.
7. Clic a **OK**.

![alt text](<picsf2/Captura de pantalla 2026-06-01 202856.png>)

---

### 4.2 Quota FSRM a la carpeta Public (per carpeta)

#### 4.2.1 Objectiu

Aplicar una quota estricta (**Hard Quota**) de 200 MB específicament a la carpeta `C:\Public`, amb un avís personalitzat quan s'arribi al 90%.

#### 4.2.2 Procediment detallat

1. Instal·lar el rol **File Server Resource Manager** (si no s'ha fet prèviament):
   - Server Manager > Manage > Add Roles and Features > File and Storage Services > File Server Resource Manager.

2. Obrir el FSRM:
   - `Windows + R` > `fsrm.msc` > **Enter**.

3. Al panell esquerre, desplegar **Quota Management** > **Quotas**.
4. Clic dret > **Create Quota...**
5. A **Quota path**, introduir:
   ```
   C:\Public
   ```
6. Seleccionar **Define custom quota properties** > Clic a **Custom Properties...**

7. A la finestra de propietats:
   - **Space limit:** `200` MB
   - Seleccionar **Hard quota** (bloca l'escriptura quan s'arriba al límit).

8. Afegir un llindar d'avís (Threshold):
   - Clic a **Add** > **Generate notifications when usage reaches:** `90%`.
   - A la pestanya **E-mail Message** (o **Event Log** si no hi ha SMTP configurat), escriure el missatge:
     > "Compte! FoodLogístic t'informa que estàs a punt d'esgotar l'espai compartit."

9. Clic a **OK** > **Create**.

![alt text](<picsf2/Captura de pantalla 2026-06-01 202930.png>)

![alt text](<picsf2/Captura de pantalla 2026-06-01 202930.png>)

---

### 4.3 Filtrat de fitxers (File Screen) a Operacions

#### 4.3.1 Objectiu

Impedir que els usuaris del grup Transport puguin guardar fitxers executables, d'àudio, vídeo i altres extensions no autoritzades dins `C:\Operacions`.

#### 4.3.2 Raonament

FSRM permet crear **pantalles de fitxers** (*File Screens*) que actuen a nivell de carpeta. A diferència del filtratge per extensió bàsic, FSRM pot analitzar la signatura interna del fitxer, de manera que canviar l'extensió no eludeix el sistema.

#### 4.3.3 Procediment detallat

**Pas 1: Crear un Grup de Fitxers personalitzat**

1. Dins el **FSRM**, desplegar **File Screening Management** > **File Groups**.
2. Clic dret > **Create File Group...**
3. **File group name:** `Extensions Prohibides Extra`
4. A **Files to include**, afegir una a una:
   - `*.bat`
   - `*.cmd`
   - `*.mkv`
   - `*.mov`
5. Clic a **OK**.

**Pas 2: Crear el File Screen**

1. Anar a **File Screening Management** > **File Screens**.
2. Clic dret > **Create File Screen...**
3. **File screen path:** `C:\Operacions`
4. Seleccionar **Define custom file screen properties** > **Custom Properties...**
5. **Screening type:** seleccionar obligatòriament **Active screening** (bloca realment el fitxer; el *Passive* només registra).
6. A **File groups**, marcar:
   - **Executable Files**
   - **Audio and Video Files**
   - **Extensions Prohibides Extra**

![alt text](<picsf2/Captura de pantalla 2026-06-01 202948.png>)

7. (Opcional) A la pestanya **Actions**, afegir un missatge al registre d'esdeveniments.
8. Clic a **OK** > **Create**.

![alt text](<picsf2/Captura de pantalla 2026-06-01 203057.png>)

![alt text](<picsf2/Captura de pantalla 2026-06-01 203112.png>)

#### 4.3.4 Anàlisi tècnica

El filtratge actiu de FSRM analitza la **signatura interna** del fitxer (els seus *magic numbers*), no només l'extensió del nom. Per tant, si un usuari canvia `virus.exe` a `virus.txt`, el sistema continuarà bloquejant-lo perquè detecta que és un executable.

---

## 5. Proves de Funcionament (des de client Windows 10/11)

Per verificar que tota la infraestructura funciona correctament, s'han realitzat les següents comprovacions des d'un client unit al domini:

### 5.1 Verificació d'accés i visibilitat per perfil d'usuari

| Tipus d'usuari | Carpetes visibles | Unitat C: | Pot escriure? |
|----------------|-------------------|-----------|---------------|
| **Transport** | Veu `Public` i `Operacions` | No apareix | Sí a `Operacions`, només lectura a `Public` |
| **Direccio** | Veu `Public`, `Operacions` i `Direccio` | Sí (C:) | Sí a `Direccio`, lectura a `Public` |
| **Administracio** | Veu `Public` i `Operacions` (no veu `Direccio` per ABE) | No apareix | Només lectura a `Public` |

La carpeta `Direccio` no apareix als usuaris que no en formen part gràcies a l'**ABE** configurat tant al Server Manager com a PowerShell.

### 5.2 Prova de filtratge de fitxers (Operacions)

- **Intent 1:** Copiar `notepad.exe` a `\\FL11\Operacions` > **Bloquejat per FSRM**.
- **Intent 2:** Renombrar `notepad.exe` a `notepad.txt` i copiar-lo > **Continua bloquejat**.  
  Això demostra que el filtratge actiu no es deixa enganyar pel canvi d'extensió.

### 5.3 Prova de quota (Public)

- Omplir la carpeta `Public` fins a 180 MB (90% de 200 MB).
- Resultat: Apareix l'avís personalitzat configurat al FSRM.
- Intentar superar els 200 MB: el sistema denega l'escriptura amb error d'**espai insuficient** (comportament de *Hard Quota*).

---

## 6. Conclusions i Millores Aplicades

S'ha aconseguit implementar una infraestructura de fitxers **segura, organitzada i controlada** per a FoodLogistic, complint tots els requisits de l'enunciat:

- **Tres vies d'administració demostrades:** Explorador de fitxers (GUI bàsica), Server Manager (GUI avançada) i PowerShell (línia de comandes/scripting).
- **Seguretat per capes:** combinació òptima de permisos SMB + NTFS + ABE + GPOs filtrades.
- **Control d'espai:** quotes NTFS per volum (500 MB/usuari) i quotes FSRM per carpeta (200 MB a Public amb avís al 90%).
- **Prevenció de contingut no desitjat:** File Screen actiu a `Operacions` bloquejant executables i multimèdia.

### Recomanacions futures

- Implantar **DFS Namespace** per unificar els camins UNC i facilitar la migració futura de servidors.
- Implementar **Dynamic Access Control (DAC)** per a una gestió més granular basada en atributs d'usuari.
- Configurar **Shadow Copies** (còpies d'ombra) per permetre la recuperació de fitxers per part dels usuaris sense intervenció de l'administrador.
"""


