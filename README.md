<div dir="rtl" align="right">

# Holiday Events – CI/CD Project

פרויקט מסכם בקורס DevOps: בניית תהליך CI/CD מלא לאפליקציית Node.js קיימת בשם **Holiday Events**, כולל Git, Docker, Jenkins, Ansible, וחיבור כל השלבים לתהליך אוטומטי אחד.

קוד האפליקציה המקורי: https://github.com/almayomekonen/holiday

---

## מבנה הפרויקט

```
holiday/
├── ansible/
│   ├── deploy.yml        # Ansible Playbook - פריסת האפליקציה כ-container
│   └── inventory         # רשימת שרתי היעד
├── data/
│   └── events.json       # נתוני האפליקציה
├── public/                # קבצי frontend סטטיים
├── Dockerfile             # הגדרת ה-image של האפליקציה
├── Jenkinsfile             # הגדרת ה-Pipeline המלא
├── package.json
├── server.js               # שרת Express הראשי
└── README.md
```

---

## איך ה-Pipeline עובד

ה-Jenkinsfile מגדיר Pipeline עם השלבים הבאים, שרצים אוטומטית בכל push ל-branch `main`:

1. **Checkout** — משיכת הקוד העדכני מ-GitHub
2. **Extract Commit SHA** — שמירת ה-commit hash הנוכחי לצורך תיעוד
3. **Install & Test** — התקנת Node.js מקומית (ב-workspace, ללא הרשאות root) והרצת `npm install`
4. **Build Docker Image** — בניית image עם tag כפול: מספר ה-build הנוכחי ו-`latest`
5. **Push to Docker Hub** — התחברות והעלאת ה-image ל-Docker Hub (`wubdigitalinfo/holiday-app`)
6. **Deploy with Ansible** — הרצת Ansible Playbook שמתחבר לשרת יעד נפרד, מוריד את ה-image החדש, ומריץ אותו כ-container
7. **Health Check** — בדיקת `/health` על השרת המרוחק. אם התקין — נרשם כ-build "ידוע כתקין" לצורך rollback עתידי. אם נכשל — מתבצע **rollback אוטומטי** לגרסה התקינה הקודמת, וה-pipeline מסתיים ב-failure (כדי שהכשל יהיה גלוי, גם אם ה-rollback הצליח)

הטריגר האוטומטי מוגדר ב-Jenkins כ-**Scan Multibranch Pipeline Trigger** (polling כל דקה) — כל push חדש מזוהה ומריץ build לבד, בלי לחיצה ידנית.

---

## אילו מכונות/שרתים נדרשים

| מכונה                                          | תפקיד                                                                                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Jenkins container** (`jenkins-server`)       | מריץ את כל שלבי ה-CI: checkout, build, push. מכיל Docker CLI (עם גישה ל-Docker socket של ה-host), Ansible, ו-GitHub CLI (`gh`) |
| **שרת יעד נפרד** (Codespace `humble-chainsaw`) | שרת ה-deployment. מריץ את קונטיינר האפליקציה בפועל, נגיש ל-Ansible דרך SSH                                                     |
| **Docker Hub** (`wubdigitalinfo/holiday-app`)  | Container Registry — מאגר ה-images שביניהם עוברת האפליקציה                                                                     |

**חיבור בין השרתים:** קונטיינר Jenkins מתחבר לשרת היעד באמצעות SSH, שמנותב דרך `gh codespace ssh` (GitHub CLI). הגדרת ה-SSH נוצרה עם:

```bash
gh codespace ssh --config -c <codespace-name> > ~/.ssh/config
```

זה מוסיף ל-`~/.ssh/config` (בתוך קונטיינר Jenkins) host alias בשם `cs.<codespace-name>.main`, שה-Ansible inventory מפנה אליו.

---

## איך מתבצע ה-Deployment

Ansible (`ansible/deploy.yml`) מתחבר לשרת היעד (`ansible/inventory`) ומבצע:

1. Pull של ה-Docker image העדכני מ-Docker Hub
2. עצירה והסרה של הקונטיינר הישן (אם קיים)
3. הרצת קונטיינר חדש עם `restart_policy: always`, חשוף בפורט 3000

```yaml
- hosts: webservers
  vars:
    image_name: "wubdigitalinfo/holiday-app"
  tasks:
    - name: Pull latest Docker image
      docker_image:
        name: "{{ image_name }}"
        tag: "{{ image_tag | default('latest') }}"
        source: pull

    - name: Stop and remove old container if it exists
      docker_container:
        name: holiday-container
        state: absent

    - name: Run new container
      docker_container:
        name: holiday-container
        image: "{{ image_name }}:{{ image_tag | default('latest') }}"
        state: started
        restart_policy: always
        ports:
          - "3000:3000"
```

---

## על איזה Port האפליקציה רצה

האפליקציה רצה על **פורט 3000** (מוגדר ב-`server.js` דרך `process.env.PORT`, וב-Dockerfile עם `EXPOSE 3000`). ה-Ansible playbook ממפה אותו כ-`3000:3000` על השרת המרוחק.

Endpoint לבדיקת תקינות: `GET /health` — מחזיר:

```json
{ "status": "healthy", "service": "holiday-events" }
```

---

## הגדרות חשובות להרצה מחדש

### קונטיינר Jenkins

```bash
docker run -d --name jenkins-server \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  --group-add $(stat -c '%g' /var/run/docker.sock) \
  --restart unless-stopped \
  jenkins/jenkins:lts
```

ה-`--restart unless-stopped` מבטיח שהקונטיינר יעלה מחדש אוטומטית (עם כל ההגדרות) אם הסביבה מופעלת מחדש. **חשוב:** אין להריץ שוב `docker run` אם הקונטיינר קיים כבר — משתמשים ב-`docker start jenkins-server` בלבד.

### תלויות שיש להתקין בתוך קונטיינר Jenkins (אם נוצר מחדש)

```bash
apt-get update && apt-get install -y ansible curl gnupg
# GitHub CLI:
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list
apt-get update && apt-get install -y gh
gh auth login --with-token   # עם Personal Access Token (classic) בהרשאות repo + codespace + read:org
gh codespace ssh --config -c <codespace-name> > ~/.ssh/config
chmod 600 ~/.ssh/config
```

### Credentials הנדרשים ב-Jenkins

| ID                      | סוג                                | שימוש             |
| ----------------------- | ---------------------------------- | ----------------- |
| `github-token-new`      | Username + Password (טוקן)         | Checkout מ-GitHub |
| `dockerhub-credentials` | Username + Password (Access Token) | Push ל-Docker Hub |

---

## Bonus: Automatic Rollback

מומש מנגנון **Rollback אוטומטי**: אם ה-Health Check נכשל אחרי deployment, ה-Jenkinsfile:

1. קורא את מספר ה-build האחרון שהוכר כ"תקין" (נשמר בקובץ `/tmp/last_good_build.txt` על השרת המרוחק)
2. מריץ שוב את ה-Ansible playbook עם ה-tag המדויק הזה
3. מבצע בדיקת health נוספת לאישור שה-rollback הצליח
4. מסמן את ה-pipeline כ-`FAILURE` (כדי שהכשל המקורי יהיה גלוי ולא "מוסתר")

נבדק בפועל ואומת עם endpoint מדומה — ראה `screenshots/08_automatic_rollback_proof.txt`.

---

## צילומי מסך

תיקיית `screenshots/` מכילה הוכחות ל:

1. `01_pipeline_success_console.png` — הרצה מוצלחת מקצה לקצה
2. `02_pipeline_stages_overview.png` — תצוגת השלבים
3. `03_automatic_trigger_proof.png` — הוכחת טריגר אוטומטי (ללא לחיצה ידנית)
4. `04_dockerhub_image_tags.png` — ה-images ב-Docker Hub
5. `05_github_feature_branch_merge.png` — Git branch workflow
6. `06_health_check_verified.png` — health check מוצלח
7. `07_jenkins_trigger_config.png` — הגדרת הטריגר האוטומטי
8. `08_automatic_rollback_proof.txt` — הוכחת מנגנון ה-rollback
9. `09_multibranch_overview.png` — תצוגת ה-branches ב-Jenkins

</div>
