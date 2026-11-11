# תשובות - שיעור 5

## חלק א׳: מפת ה-Cluster

| רכיב | איפה פועל | תפקיד | אנלוגיה |
|------|-----------|--------|---------|
| API server | הנהלה | מקבל את הבקשות | מזכירות |
| etcd | הנהלה | שומר את המידע | ארכיון |
| scheduler | הנהלה | בוחר איפה להריץ Pods | רכז שיבוץ |
| controller manager | הנהלה | מתאים מצב בפועל למצב הרצוי | מחנך |
| kubelet | כל Node | אחראי על ה-Pods ב-Node | מורה בכיתה |
| container runtime | כל Node | מפעיל Containers | ציוד הכיתה |
| kube-proxy | כל Node | מנתב תעבורת רשת ל-Services | שוער |
| coredns | כל Node (כ-Pods) | שמות במקום כתובות IP | ספר טלפונים |

## חלק ב׳: תרחישים

### 1. Scale

כשמשנים את מספר ה-Pods מ-2 ל-4, הבקשה מגיעה ל-API server, שמעדכן את המצב הרצוי ב-etcd. ה-controller manager מזהה שיש רק 2 Pods במקום 4 ויוצר 2 Pods חדשים. ה-scheduler בוחר לכל Pod חדש Node מתאים. ה-kubelet באותו Node מפעיל את ה-Pod, וה-container runtime מריץ את ה-Container.

### 2. שלוש בלילה

כש-Backend קורס, ה-kubelet ב-Node מזהה זאת ומדווח ל-API server. ה-controller manager מגלה שמספר ה-Pods בפועל נמוך מהמצב הרצוי, ויוצר Pod חדש במקום זה שקרס. ה-scheduler בוחר לו Node, וה-kubelet מפעיל אותו. כך האפליקציה מתאוששת אוטומטית, בלי התערבות אנושית.

אם מישהי הייתה מוחקת את ה-Pod כולו, התוצאה הייתה דומה: ה-controller manager מזהה שיש פחות Pods מהרצוי ויוצר Pod חדש, וה-scheduler וה-kubelet מפעילים אותו. את זה גם ראיתי בטרמינל כשמחקתי את ה-Pods של greeter והם חזרו לבד.

### 3. Node נכבה

כש-Node נכבה, ה-controller manager מזהה שהוא הפסיק לדווח ומסמן אותו כלא זמין. ה-Pods שהיו עליו נחשבים אבודים, ולכן מספר ה-Pods בפועל נמוך מהמצב הרצוי. ה-controller manager יוצר Pods חדשים, וה-scheduler משבץ אותם על שני ה-Nodes שנשארו. ה-kubelet בכל Node מפעיל אותם, והאפליקציה ממשיכה לעבוד.

ב-cluster של Docker Desktop בכיתה יש Node אחד בלבד, ולכן אם המחשב נכבה, כל ה-cluster נעצר: אין Node אחר שאליו אפשר להעביר את ה-Pods. כשהמחשב עולה שוב, ה-cluster עולה והרכיבים מחזירים את ה-Pods למצב הרצוי.

### 4. מהכיתה לענן

בקבצי ה-YAML מהכיתה יש `imagePullPolicy: Never`, שאומר "אל תוריד Image, השתמש במה שקיים במחשב". ב-EKS ה-Nodes הם מחשבים אחרים שאין בהם את ה-Image שבניתי מקומית, ולכן ה-Pods ייתקעו ב-ErrImageNeverPull. צריך להעלות את ה-Image למאגר כמו ECR, לכתוב ב-YAML את הכתובת המלאה שלו, ולהסיר את `Never`. בנוסף, Service מסוג `NodePort` נותן גישה דרך פורט על ה-Nodes, שבענן בדרך כלל לא פתוחים לעולם, ולכן משתמשים ב-`LoadBalancer` שמקבל כתובת ציבורית.

### 5. Pod בודד

Pod שנוצר באופן עצמאי לא שייך לשום controller, ולכן אף רכיב לא בודק שהוא קיים, וכשמוחקים אותו הוא נעלם לצמיתות. Pod ששייך ל-Deployment נמצא תחת פיקוח של ReplicaSet, שמגדיר כמה עותקים צריכים להיות. כשמוחקים Pod כזה, ה-controller manager מזהה שהמצב בפועל שונה מהרצוי ויוצר Pod חדש. ה-scheduler משבץ אותו ל-Node, וה-kubelet מפעיל אותו.

כדי ש-stats יחזור תמיד, הייתי עוטפת אותו ב-Deployment עם `replicas` של 1 לפחות, במקום ליצור אותו עם `kubectl run`. ככה ה-Deployment ידאג שתמיד יהיה Pod כזה, ואם הוא יימחק או ייפול, ייווצר חדש.