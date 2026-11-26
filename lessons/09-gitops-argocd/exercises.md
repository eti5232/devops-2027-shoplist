# שיעור 9 — תרגולים לבית

**לא מוגש.** התרגולים מכינים לשיעורי הבית. תרגולים 1–4 ו-6 על הנייר. תרגול 5 על המחשב, אחרי שהתקנתן את Argo CD (ד1–ד3 בשיעורי הבית).

**איך לעבוד:** לנסות לבד. נתקעתן? לפתוח את **הפתרון** מתחת לתרגיל. אבל קודם לנסות, גם אם לא בטוחות.

---

## תרגול 1 — push או pull (על הנייר, 10 דקות)

לכל מצב, לכתוב: push או pull? ולמה?

1. צעד ב-GitHub Actions מריץ `helm upgrade` מול cluster בענן, עם קובץ kubeconfig ששמור ב-Secrets.
2. הכלי Argo CD, בתוך ה-cluster, בודק את הריפו כל 3 דקות.
3. ה-cluster שלכן ב-Docker Desktop, בבית. ה-pipeline ב-GitHub צריך להתקין עליו.
4. מישהי מריצה `kubectl apply -f` מהמחשב שלה.
5. בחברה, ה-cluster ברשת סגורה: מותר לצאת ממנו לאינטרנט, אבל אסור להיכנס אליו מבחוץ. איזו דרך תעבוד?

<details>
<summary>פתרון</summary>

1. **זה push.** ה-pipeline מתחבר מבחוץ ל-cluster, ומחזיק מפתח אליו.
2. **זה pull.** הסוכן בתוך ה-cluster יוצא אל Git.
3. **דרך push לא תעבוד:** ל-cluster אין כתובת באינטרנט, ו-GitHub לא יכול להגיע אליו. רק pull: Argo CD בתוך ה-cluster, שיוצא אל GitHub.
4. **זה push**, ביד. ואם Argo CD מנהל את ה-cluster עם selfHeal, השינוי יחזור.
5. **רק pull.** יציאה החוצה מותרת, אז Argo CD יכול לקרוא את Git. push דורש כניסה מבחוץ, וזה אסור.

</details>

---

## תרגול 2 — לקרוא Application (על הנייר, 10 דקות)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/dana/devops-2027-shoplist.git
    targetRevision: main
    path: helm/shoplist
    helm:
      valueFiles: [values-registry.yaml, values-staging.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      selfHeal: true
```

1. מאיזה ריפו, ומאיזו תיקייה?
2. לאיזה namespace יותקן?
3. שני קובצי values. אם שניהם קובעים `frontend.nodePort`, מי מנצח?
4. דנה מוחקת את `templates/backup-cronjob.yml` מה-chart, ועושה push. מה יקרה ל-CronJob ב-cluster?
5. ה-namespace `staging` עוד לא קיים. מה יקרה בסנכרון הראשון? מה חסר?

<details>
<summary>פתרון</summary>

1. הריפו של `dana`, branch `main`, התיקייה `helm/shoplist`.
2. ל-`staging`.
3. האחרון ברשימה: `values-staging.yaml`. בדיוק כמו כמה `-f` ב-Helm.
4. הוא **נשאר** ב-cluster. אין `prune: true`, אז Argo CD לא מוחק. הוא יסמן אותו OutOfSync, עם סמל של פח.
5. הסנכרון ייכשל: `namespaces "staging" not found`. חסר `syncOptions: [CreateNamespace=true]`.

</details>

---

## תרגול 3 — מצאו את הטעות (על הנייר, 15 דקות)

בכל קטע יש טעות אחת. מה היא, ומה יקרה?

**א.**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shoplist
  namespace: shop
spec:
  project: default
  source:
    repoURL: https://github.com/dana/devops-2027-shoplist.git
    targetRevision: main
    path: helm/shoplist
  destination:
    server: https://kubernetes.default.svc
    namespace: shop
```

**ב.**

```yaml
  source:
    repoURL: https://github.com/dana/devops-2027-shoplist.git
    targetRevision: main
    path: helm/shoplist
    helm:
      valueFiles: [helm/shoplist/values-registry.yaml]
```

**ג.** job של deploy ב-`ci.yml`:

```yaml
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v6
      - run: |
          sed -i -E "s/^( +tag:).*/\1 \"${GITHUB_SHA::7}\"/" helm/shoplist/values-registry.yaml
          git config user.name "github-actions[bot]"
          git config user.email "github-actions@github.com"
          git commit -am "Deploy" && git push
```

**ד.** ה-job של deploy, עם ה-permissions של ה-workflow משיעור 8, ובלי `permissions` ב-job:

```yaml
permissions:
  contents: read
  packages: write
```

<details>
<summary>פתרון</summary>

**א.** `metadata.namespace: shop`. Argo CD קורא Applications רק מה-namespace שלו, `argocd`. ה-Application נוצר, אבל אף אחד לא מסתכל עליו: אין סטטוס, ושום דבר לא קורה.

**ב.** הנתיב ב-`valueFiles` הוא **בתוך** התיקייה של ה-chart. צריך `values-registry.yaml` בלבד. כאן Argo CD יחפש `helm/shoplist/helm/shoplist/values-registry.yaml`, וייכשל עם `ComparisonError`.

**ג.** חסר `needs: build`. ה-job רץ במקביל ל-`test` ול-`build`, כותב תגית חדשה ל-Git מיד, ו-Argo CD מנסה למשוך image שעוד לא קיים: `ImagePullBackOff`. ואם `test` נכשל, התגית כבר ב-Git.

**ד.** `contents: read` לא מתיר push. ה-`git push` ייכשל עם `Permission denied` או `403`. צריך `permissions: contents: write` ב-job של deploy.

</details>

---

## תרגול 4 — Synced ו-Healthy (על הנייר, 10 דקות)

לכל מצב: מה ה-Sync status, ומה ה-Health status? Argo CD עם `automated` ו-`selfHeal`, אלא אם כתוב אחרת.

1. הכל רץ, כמו שכתוב ב-Git.
2. ב-Git שיניתן תגית ל-`zzz9999`, שלא קיימת ב-ghcr. Argo CD סנכרן.
3. יש Application חדש, בלי `automated`. עוד לא לחצתן SYNC.
4. בלי `selfHeal`. מישהי הריצה `kubectl scale ... --replicas=3`.
5. עשיתן push של שינוי, ו-Argo CD עוד לא בדק את Git.
6. אחרי SYNC, ה-pods עוד עולים, ה-image נמשך.

<details>
<summary>פתרון</summary>

1. **סטטוס: Synced, Healthy.**
2. **סטטוס: Synced, Degraded** (אחרי זמן; קודם Progressing). ה-cluster בדיוק כמו Git, אבל ה-pod ב-`ImagePullBackOff`.
3. **סטטוס: OutOfSync, Missing.**
4. **סטטוס: OutOfSync, Healthy.** הכל עובד, אבל לא מה שכתוב. בלי selfHeal, זה נשאר כך.
5. **סטטוס: Synced, Healthy.** Argo CD עוד לא יודע על השינוי. אחרי REFRESH, או בסיבוב הבא: OutOfSync, ומיד סנכרון.
6. **סטטוס: Synced, Progressing.**

</details>

---

## תרגול 5 — לשבור ב-cluster (על המחשב, 20 דקות)

**צריך:** ShopList ב-Argo CD, Synced ו-Healthy (ד3 בשיעורי הבית).

1. ב-`argocd/application.yaml`, למחוק את השורה `selfHeal: true`. `kubectl apply`.
2. להריץ `kubectl scale deployment stats -n shop --replicas=3`. מה ה-Sync status? כמה pods של stats אחרי דקה?
3. ב-DIFF: מה ההבדל שמוצג?
4. ללחוץ SYNC. מה קורה ל-stats?
5. להחזיר את `selfHeal: true`, ו-`kubectl apply`. שוב `kubectl scale` ל-3. מה קורה עכשיו?
6. להריץ `kubectl delete service stats -n shop`. מה קורה?

<details>
<summary>פתרון</summary>

2. **סטטוס: OutOfSync.** שלושה pods של stats נשארים: בלי selfHeal, Argo CD רק מראה.
3. ב-Deployment של stats: `replicas: 3` ב-cluster, מול `replicas: 1` ב-Git.
4. הכפתור SYNC מחזיר ל-1. שני pods נמחקים.
5. תוך שניות, חוזר ל-1. Synced.
6. ה-Service נוצר מחדש תוך שניות. selfHeal מחזיר גם משאב שנמחק, לא רק שינוי.

</details>

---

## תרגול 6 — לתרגם ל-GitLab CI (על הנייר, 15 דקות)

הנה job מ-GitHub Actions:

```yaml
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: "20"
      - name: Stats - unit tests
        working-directory: app/stats
        run: npm test
```

לכתוב את אותו job ב-`.gitlab-ci.yml`, בשלב `test`. ואז לענות:

1. מה קרה ל-`checkout`?
2. מה קרה ל-`setup-node`?
3. איך GitLab יודע לא להתחיל את `build` לפני ש-`test` הצליח?

<details>
<summary>פתרון</summary>

```yaml
stages: [test, build]

test-stats:
  stage: test
  image: node:20-alpine
  script:
    - cd app/stats
    - npm test
```

1. אין צורך. GitLab מביא את הקוד לבד לכל job.
2. במקום להתקין Node, בוחרות `image` שכבר יש בו Node 20.
3. בגלל `stages`: השלב `build` מתחיל רק אחרי שכל ה-jobs בשלב `test` הצליחו.

</details>
