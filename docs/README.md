# 🖥️ Desktop Automation — WinAppDriver + Robot Framework

Tester une application .NET (WinForms, WPF, UWP) automatiquement, c'est possible
avec WinAppDriver — le service Microsoft qui expose une API WebDriver pour les
applications Windows. Combiné avec Robot Framework et DesktopLibrary, on obtient
un framework de test desktop lisible, maintenable et intégrable en CI/CD.

---

## Architecture du setup

```
  Application .NET testée (WinForms / WPF / UWP)
          │
          │  UIAutomation API (Windows)
          ▼
  ┌───────────────────────────────────────────────┐
  │              WinAppDriver.exe                 │
  │         (écoute sur port 4723)                │
  │                                               │
  │  Expose une API WebDriver/Appium compatible   │
  │  pour interagir avec les éléments UI Windows  │
  └────────────────────┬──────────────────────────┘
                       │  HTTP / JSON Wire Protocol
                       ▼
  ┌───────────────────────────────────────────────┐
  │         Robot Framework + DesktopLibrary      │
  │                                               │
  │  Mots-clés : Open Application, Click Element  │
  │  Input Text, Get Text, Select From ComboBox   │
  │  Wait Until Element Is Visible, Close App...  │
  └───────────────────────────────────────────────┘
```

---

## Identifier les éléments UI — Accessibility Insights

Avant d'écrire un test, il faut trouver les locators des éléments.
**Accessibility Insights** (outil Microsoft gratuit) inspecte chaque contrôle :

```
  Élément : Bouton "Enregistrer"
  ─────────────────────────────────────────────
  AutomationId  : btn_save          ← le plus stable
  Name          : Enregistrer       ← peut changer si i18n
  ClassName     : Button
  XPath         : //Button[@AutomationId='btn_save']
  ─────────────────────────────────────────────
  Stratégie recommandée : AutomationId > Name > XPath
```

---

## Suite de tests Robot Framework — application desktop .NET

```robotframework
*** Settings ***
Library     AppiumLibrary
Library     DesktopLibrary
Suite Setup     Start WinAppDriver
Suite Teardown  Stop WinAppDriver

*** Variables ***
${APP_PATH}     C:\\Program Files\\MonApp\\MonApp.exe
${REMOTE_URL}   http://127.0.0.1:4723

*** Test Cases ***

Ouverture de l application
    [Documentation]    Vérifie que l'application démarre et affiche l'écran principal
    Open Application    ${REMOTE_URL}
    ...    app=${APP_PATH}
    ...    platformName=Windows
    ...    deviceName=WindowsPC
    Wait Until Element Is Visible    accessibility_id=main_window    timeout=10s
    Element Should Be Visible        accessibility_id=menu_bar

Login avec identifiants valides
    [Documentation]    Vérifie le flux de connexion complet
    Click Element           accessibility_id=btn_login
    Input Text              accessibility_id=field_username    admin
    Input Text              accessibility_id=field_password    secret123
    Click Element           accessibility_id=btn_submit
    Wait Until Element Is Visible    accessibility_id=dashboard    timeout=5s
    Element Text Should Be  accessibility_id=welcome_label    Bienvenue, admin

Saisie dans un formulaire
    [Documentation]    Teste la saisie et validation d'un formulaire
    Click Element           accessibility_id=menu_nouveau
    Select Elements From Menu   accessibility_id=menu_fichier
    ...                         accessibility_id=menu_nouveau_document
    Input Text              accessibility_id=field_titre    Test Document
    Select Element From ComboBox
    ...    accessibility_id=combo_categorie
    ...    accessibility_id=item_rapport
    Click Element           accessibility_id=btn_enregistrer
    Element Should Be Visible   accessibility_id=notification_succes

Détection d une anomalie — champ obligatoire vide
    [Documentation]    Vérifie qu'une erreur s'affiche si le champ titre est vide
    Click Element           accessibility_id=btn_enregistrer
    Element Should Be Visible   accessibility_id=error_titre_requis
    Element Text Should Be      accessibility_id=error_titre_requis
    ...    Le titre est obligatoire

*** Keywords ***
Start WinAppDriver
    [Documentation]    Démarre le service WinAppDriver en arrière-plan
    Start Process    WinAppDriver.exe    cwd=C:\\Program Files (x86)\\Windows Application Driver
    Sleep    2s    # Attendre que le service soit prêt

Stop WinAppDriver
    Close All Applications
    Terminate All Processes
```

---

## Sikuli — tests basés sur l'image

Pour les éléments sans AutomationId accessibles (contrôles tiers, zones graphiques),
Sikuli localise par reconnaissance d'image :

```python
# custom_library/SikuliKeywords.py
from robot.api.deco import keyword
import pyautogui
import cv2
import numpy as np

class SikuliKeywords:

    @keyword("Click Image")
    def click_image(self, image_path: str, confidence: float = 0.9):
        """
        Clique sur l'élément correspondant à l'image fournie.
        Utile pour les contrôles sans AutomationId.
        """
        location = pyautogui.locateOnScreen(image_path, confidence=confidence)
        if location is None:
            raise AssertionError(f"Image non trouvée : {image_path}")
        center = pyautogui.center(location)
        pyautogui.click(center)

    @keyword("Image Should Be Visible")
    def image_should_be_visible(self, image_path: str):
        """Vérifie qu'un élément visuel est présent à l'écran."""
        location = pyautogui.locateOnScreen(image_path, confidence=0.85)
        if location is None:
            raise AssertionError(f"Élément visuel absent : {image_path}")
```

---

## Documentation de cas de test manuel (template)

```
ID          : TC-LOGIN-001
Titre       : Connexion avec identifiants valides
Module      : Authentification
Priorité    : HAUTE
Prérequis   : Application lancée, écran de login affiché

Étapes :
  1. Saisir "admin" dans le champ Nom d'utilisateur
  2. Saisir "password123" dans le champ Mot de passe
  3. Cliquer sur le bouton "Se connecter"

Résultat attendu :
  - Redirection vers le tableau de bord
  - Message "Bienvenue, admin" affiché
  - Aucun message d'erreur visible

Résultat obtenu : [À remplir]
Statut       : [ ] PASS   [ ] FAIL   [ ] BLOCKED
Anomalie     : [Numéro si FAIL]
```

---

## Ce que j'ai appris

La stratégie de localisation des éléments UI est critique pour la stabilité des tests.
`AutomationId` est le locator le plus robuste — il ne change pas quand l'interface
est redessinée ou traduite. `Name` casse dès qu'on change le label d'un bouton.
`XPath` sur des contrôles Windows peut être extrêmement fragile si la hiérarchie
de l'arbre UI change entre deux versions de l'application.

La règle : toujours demander aux développeurs de l'application testée d'ajouter
des `AutomationId` explicites sur tous les contrôles importants. C'est 5 minutes
de travail pour eux, et ça évite des heures de maintenance des tests.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
