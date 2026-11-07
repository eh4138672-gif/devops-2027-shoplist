# שיעור 4 — תשובות

**1. ל-backend אין `ports`. איך stats מגיע אליו? ואיך הדפדפן מגיע אליו?**
stats נמצא באותה רשת ש-Compose יצר, ולכן הוא פונה ל-`http://backend:5000` לפי השם, וה-DNS של Docker מתרגם את השם לכתובת של ה-container. הדפדפן לא מגיע ל-backend ישירות: הוא פונה ל-`localhost:8080/api/...`, ו-Nginx ב-frontend (היחיד עם `ports`) מעביר את הבקשה ל-backend בתוך הרשת.

**2. `down` ואחר כך `up`: המוצרים נשארו? ומה עם `down -v`?**
אחרי `down` ו-`up` המוצרים נשארים, כי הם שמורים ב-named volume (`postgres-data`), ו-`down` מוחק רק containers ורשת, לא volumes. `down -v` מוחק גם את ה-named volume, ולכן המסד נוצר מחדש ריק (ו-`init.sql` רץ שוב).

**3. `../sql/init.sql:/docker-entrypoint-initdb.d/init.sql`: איזה סוג volume? מה יקרה אם נשנה את `init.sql` אחרי שהמסד נוצר?**
זה bind mount: הצד השמאלי הוא נתיב בקובץ במחשב (מתחיל ב-`../`), ולא שם של volume. אם משנים את `init.sql` אחרי שהמסד כבר נוצר, לא קורה כלום: postgres מריץ את הקבצים ב-`docker-entrypoint-initdb.d` רק כשתיקיית הנתונים ריקה, כלומר בפעם הראשונה. כדי שהשינוי ייכנס צריך `down -v` ו-`up`.

**4. מה שם הרשת ומה שם ה-volume?**
הרשת: `docker_default`. ה-volume: `docker_postgres-data`. Compose מוסיף לכל שם את שם הפרויקט, שהוא כברירת מחדל שם התיקייה שבה נמצא קובץ ה-Compose (`docker`). לרשת שהוא יוצר לבד הוא קורא `default`, ול-volume הוא נותן את השם שכתבנו תחת `volumes:`.

**5. מה יש ב-image הסופי של `Dockerfile.multistage` ומה אין? למה אין `CMD`? ולמה ל-stats לא צריך multi-stage?**
ב-image הסופי יש רק Nginx, את `nginx.conf` ואת קבצי התיקייה `dist` שנבנו. אין בו Node, אין את `build.js` ואין את קובצי המקור. אין `CMD` כי יורשים את ה-`CMD` של image הבסיס `nginx`, שכבר מפעיל את Nginx. ל-stats לא צריך multi-stage כי אין לו שלב בנייה: אין חבילות להתקין ואין קבצים לייצר, ו-Node נדרש גם בזמן הריצה.

**6. שלושת הגדלים, ולמה `multi` כמעט בגודל של `single`?**

| image | גודל |
|---|---|
| `single` | ____ |
| `multi` | ____ |
| `kitchen` | ____ |

`multi` כמעט בגודל של `single` כי שניהם מבוססים על אותו `nginx:1.30-alpine` ומכילים כמעט אותם קבצים (כמה KB של אתר). כל Node נשאר בשלב `build`, שלא נכנס ל-image הסופי. `kitchen` גדול בהרבה כי הוא כולל את כל Node.

**7. למה `/api/stats` מגיע ל-stats? ומה קורה ל-`/api/products` בגלל הלוכסן בסוף `http://backend:5000/`?**
Nginx בוחר את ה-`location` עם הקידומת הארוכה ביותר שמתאימה. `/api/stats` ארוך יותר מ-`/api/`, ולכן הוא מנצח והבקשה הולכת ל-stats. הלוכסן בסוף `proxy_pass` גורם ל-Nginx להחליף את החלק שהתאים (`/api/`) בכתובת שאחריו, ולכן `/api/products` מגיע ל-backend כ-`/products`, וזה הנתיב שה-backend מכיר.
