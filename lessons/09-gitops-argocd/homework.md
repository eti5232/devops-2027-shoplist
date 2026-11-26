# שיעור 9 — שיעורי בית: GitOps ל-ShopList

## מסמך דרישות

**המטרה:** Argo CD ב-cluster שלכן, שמסתכל על הריפו שלכן ב-GitHub. כל שינוי מגיע ל-cluster רק דרך Git. ובסוף: push של קוד מגיע לבד עד הדפדפן, דרך ה-pipeline משיעור 8 ו-Argo CD.

**הגשה:** push מהבית, לפני שיעור 10.

**זמן משוער:** שעתיים וחצי. עשיתן קודם את [התרגולים](exercises.md)? פחות.

**איך לעבוד:** הדרישות מסודרות לפי הסדר. כל דרישה נשענת על הקודמת. בכל דרישה כתוב **איך בודקות שזה עובד**. לא עוברות לדרישה הבאה לפני שהבדיקה עברה.

---

## 0. הכנה (פעם אחת)

### מה צריך

- [ ] שיעורי הבית של שיעור 8: `ci.yml` ירוק, ושלושה packages **ציבוריים** ב-`ghcr.io`
- [ ] בריפו: `helm/shoplist/values-registry.yaml`, עם השם שלכן במקום `OWNER`
- [ ] זיכרון ל-Docker Desktop: Settings ← Resources ← Memory: **6GB לפחות**
- [ ] ב-Docker Desktop: Kubernetes running, ו-`kubectl config current-context` מחזיר `docker-desktop`
- [ ] הפורט 30080 פנוי: ShopList משיעור 8 לא תופס אותו. את 30080: `helm uninstall shop -n shop`

### להביא את הקוד

בתיקייה של הריפו שלכן: `git pull`.

---

## 1. התמונה הגדולה: מה יהיה בסוף

```
argocd/
  application.yaml         ← ד2, ד3
.github/
  workflows/
    ci.yml                 ← ד5: job חדש, deploy
helm/
  shoplist/
    values.yaml            ← ד3: commits של שינויים
    values-registry.yaml   ← ד5: התגית נכתבת לבד
lessons/09-gitops-argocd/homework/
  answers.md
  gitlab-ci.yml            ← ד6
  screenshots/
    1-synced.png
    2-diff.png
    3-selfheal.png
    4-history.png
    5-full-loop.png
```

---

## 2. הדרישות

### ד1 — Argo CD מותקן

לפי חלק 4 במדריך: `helm install`, הסיסמה, ו-`port-forward`.

**בדיקה:**
- הפקודה `kubectl get pods -n argocd`: כל ה-pods `Running`
- באתר `http://localhost:8080`: נכנסות עם `admin`, ורואות דף ריק

### ד2 — Application ראשון

1. קובץ `argocd/application.yaml`, לפי שלב 4 במדריך, **בלי** `automated`. עם הריפו שלכן ב-`repoURL`, ו-`values-registry.yaml` ב-`valueFiles`.
2. לעשות commit ו-push.
3. להריץ `kubectl apply -f argocd/application.yaml`.
4. באתר: OutOfSync ו-Missing. ללחוץ SYNC.

**בדיקה:**
- הפקודה `kubectl get applications -n argocd`: `Synced` ו-`Healthy`
- הפקודה `kubectl get pods -n shop -o wide`: הכל `Running`, וה-images מ-`ghcr.io/<השם שלכן>/...`
- האתר `http://localhost:30080` עובד
- הפקודה `helm list -n shop` מחזירה רשימה ריקה. למה? (שאלה 3 בד6)

**צילום מסך 1:** העץ של `shoplist`, עם Synced ו-Healthy למעלה.

### ד3 — לשנות דרך Git, ואוטומטי

1. ב-`values.yaml`: `stats.replicas: 2`. commit ו-push.
2. באתר: REFRESH. OutOfSync. לפתוח DIFF, עם Compact diff ו-Inline diff.
3. להוסיף ל-`application.yaml` את `automated`, עם `prune` ו-`selfHeal`. `kubectl apply`, commit ו-push.

**בדיקה:**
- באתר כתוב `Auto sync is enabled`, ו-Synced ל-commit האחרון
- הפקודה `kubectl get pods -n shop -l app=stats`: שני pods

**צילום מסך 2:** ה-DIFF משלב 2, שבו רואים `replicas: 1` מול `replicas: 2`.

### ד4 — self-heal, ו-git revert

1. להריץ `kubectl scale deployment stats -n shop --replicas=4`, ולעבור מיד לאתר.
2. ב-`git log --oneline`, למצוא את ה-commit של `stats.replicas: 2`. ואז `git revert --no-edit <המזהה שלו>`, ו-push.

**בדיקה:**
- אחרי שלב 1: תוך שניות, שוב שני pods של stats
- אחרי שלב 2: pod אחד של stats, ו-Synced ל-commit של ה-revert

**צילום מסך 3:** הטרמינל: פקודת ה-`scale`, ומיד אחריה `kubectl get deployment stats -n shop`, שמראה `2/2`.
**צילום מסך 4:** HISTORY AND ROLLBACK, עם ה-revert למעלה.

### ד5 — המעגל המלא: job של deploy

1. ב-`.github/workflows/ci.yml`, job שלישי בשם `deploy`, לפי שלב 11 במדריך:
   - רץ **רק אחרי** `build`
   - ההרשאה `permissions: contents: write`, **רק** ב-job הזה
   - כותב את 7 התווים של ה-commit לכל שורת `tag:` ב-`values-registry.yaml`, ועושה commit בשם `Deploy <תגית>`, ו-push
2. שינוי קטן בקוד: הכותרת ב-`app/frontend/index.html`, מ-`ShopList` ל-`ShopList של <השם שלכן>`. commit ו-push.

**בדיקה:**
- בלשונית Actions: `test`, שלושה `build`, ואז `deploy`. הכל ירוק
- להריץ `git pull`, ואז `git log --oneline -2`: למעלה `Deploy <תגית>`, שלא אתן עשיתן
- ב-`values-registry.yaml`: שלוש שורות `tag:` עם התגית הזו
- באתר של Argo CD: Synced ל-commit של ה-Deploy
- באתר `http://localhost:30080`, אחרי רענון: הכותרת החדשה

**צילום מסך 5:** הגרף של הריצה ב-Actions, עם `deploy` ירוק, וליד (או מתחת) הדפדפן עם הכותרת החדשה.

**מעכשיו:** לפני כל עבודה, `git pull`. ה-pipeline עושה commits לריפו שלכן.

### ד6 — שאלות, ותרגום ל-GitLab CI

**א. שאלות.** ב-`lessons/09-gitops-argocd/homework/answers.md`, תשובה של 2–4 משפטים לכל שאלה. במילים שלכן.

1. למה ה-pipeline ב-GitHub לא יכול להריץ `helm upgrade` על ה-cluster שלכן?
2. מה ההבדל בין Sync status ל-Health status? תני דוגמה למצב שהוא Synced וגם Degraded.
3. למה `helm list -n shop` ריק, למרות ש-ShopList רץ?
4. בד4 הרצת `kubectl scale`, וזה חזר. מה היה קורה בלי `selfHeal`?
5. למה חוזרים אחורה עם `git revert`, ולא עם `helm rollback` או הכפתור ב-Argo CD?
6. ה-job של deploy עושה push לריפו. למה זה לא מפעיל את ה-pipeline שוב, בלולאה?
7. למה ה-job של deploy צריך `needs: build`? מה היה קורה בלעדיו?
8. מישהי בצוות שואלת: "מה רץ עכשיו ב-cluster?". איך תעני, בלי `kubectl`?

**ב. GitLab CI.** קובץ `lessons/09-gitops-argocd/homework/gitlab-ci.yml`, על הנייר (לא מריצים): ה-pipeline שלכן משיעור 8, ב-GitLab CI. שלב `test` עם בדיקת התחביר ובדיקות היחידה של backend, ושלב `build` עם `parallel: matrix` לשלושת הסרוויסים. חלק 8 במדריך.

---

## 3. בונוס

### בונוס ד7 — שתי סביבות

עוד Application, `shoplist-staging`, מאותו ריפו ומאותו chart, ל-namespace `shop-staging`:

- קובץ `helm/shoplist/values-staging.yaml`, עם `frontend.nodePort: 30081` ו-`backup.enabled: false`
- ב-`valueFiles`: קודם `values-registry.yaml`, ואחריו `values-staging.yaml`

**בדיקה:** שני כרטיסים באתר, שניהם Synced ו-Healthy. `http://localhost:30081` עובד, ואין CronJob ב-`shop-staging`.

---

## 4. מה מגישות

- [ ] קובץ `argocd/application.yaml`, עם `automated`
- [ ] קובץ `ci.yml` עם `deploy`, ובהיסטוריה של הריפו commit בשם `Deploy ...`
- [ ] הקבצים `answers.md` ו-`gitlab-ci.yml`, וחמישה צילומי מסך
- [ ] לעשות push מהבית

---

## 5. ניקוד

| דרישה | נקודות |
|---|---|
| ד1: Argo CD מותקן | 10 |
| ד2: Application, Synced ו-Healthy | 20 |
| ד3: שינוי דרך Git, ואוטומטי | 15 |
| ד4: self-heal ו-revert | 15 |
| ד5: job של deploy, המעגל המלא | 25 |
| ד6: שאלות ו-GitLab CI | 15 |
| בונוס ד7 | +5 |

---

## 6. תקלות שכדאי להכיר

| מה רואות | מה עושים |
|---|---|
| pods של `argocd` ב-`Pending` או `OOMKilled` | חסר זיכרון. Docker Desktop: 6GB לפחות |
| `repository not found` | הריפו פרטי, או טעות ב-`repoURL` |
| `ComparisonError` על `values-registry.yaml` | הקובץ לא נדחף, או נתיב לא נכון ב-`valueFiles` (רק שם הקובץ) |
| Synced, אבל `ImagePullBackOff` | ה-package פרטי, או התגית לא קיימת ב-ghcr |
| `provided port is already allocated` | ShopList ישן תופס את 30080. `helm uninstall shop -n shop` |
| שיניתי ב-Git, ולא קורה כלום | REFRESH. ובאמת עשיתן push? |
| deploy: `Permission denied` או `403` ב-push | חסר `permissions: contents: write` ב-job |
| `! [rejected] ... (fetch first)` כשאתן עושות push | ה-pipeline עשה commit. `git pull --rebase`, ואז push |
| deploy: `nothing to commit` | התגית כבר בקובץ. הרצתן שוב את אותו commit? זה בסדר |
| אין כניסה ל-`localhost:8080` | ה-port-forward נסגר. להריץ שוב, בטרמינל נפרד |
