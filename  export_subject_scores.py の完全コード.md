

了解です。

では **Phase 1.5 の中核**となる

✅ **export_subject_scores.py の完全コード** を提示します。



このコードは、これまであなたと合意してきた **最終仕様どおり**です。



------





# **📄 export_subject_scores.py（完全版）**





**配置場所**

```
exam2/management/commands/export_subject_scores.py
```



------



```
from django.core.management.base import BaseCommand
from django.conf import settings
from django.db.models import Sum, Case, When, F, IntegerField
from pathlib import Path
import json

from exam2.models import (
    Subject,
    Exam,
    Question,
    Student,
    StudentExam,
    ExamAdjust,
    StudentExamVersion,
)


EXPORT_DIR = Path(settings.BASE_DIR) / "exam2" / "data" / "export"
EXPORT_FILE = EXPORT_DIR / "examTFdata.json"


class Command(BaseCommand):
    help = "subject 単位で StudentExam(TF/hosei) と ExamAdjust を JSON に export する"

    def add_arguments(self, parser):
        parser.add_argument("subjectNo", type=str)

    def handle(self, *args, **options):
        subjectNo = options["subjectNo"]

        subject = Subject.objects.filter(subjectNo=subjectNo).first()
        if not subject:
            self.stderr.write(self.style.ERROR("Subject が存在しません"))
            return

        fsyear = settings.FSYEAR
        term = settings.TERM

        exams = Exam.objects.filter(
            subject=subject,
            fsyear=fsyear,
            term=term
        )

        if not exams.exists():
            self.stderr.write(self.style.ERROR("Exam が存在しません"))
            return

        # 代表 exam（hash / 年度情報用）
        exam0 = exams.first()

        export_data = {
            "fsyear": fsyear,
            "term": term,
            "problem_hash": exam0.problem_hash,
            "students": {}
        }

        # 対象 StudentExamVersion
        sev_qs = StudentExamVersion.objects.filter(
            exam__in=exams
        ).select_related("student", "exam")

        self.stdout.write(f"対象学生数: {sev_qs.count()}")

        for sev in sev_qs:
            student = sev.student
            exam = sev.exam

            answers = []

            # Question は q_no 順で取得
            questions = Question.objects.filter(exam=exam).order_by("q_no")

            for q in questions:
                se = StudentExam.objects.filter(
                    student=student,
                    exam=exam,
                    question=q
                ).first()

                if not se:
                    continue

                answers.append({
                    "q_no": q.q_no,
                    "TF": se.TF,
                    "hosei": se.hosei or 0
                })

            # adjust
            adj = ExamAdjust.objects.filter(
                student=student,
                exam=exam
            ).first()

            export_data["students"][student.stdNo] = {
                "version": exam.version,
                "answers": answers,
                "adjust": adj.adjust if adj else 0
            }

        # -----------------------------
        # JSON 読み込み（既存があれば）
        # -----------------------------
        EXPORT_DIR.mkdir(parents=True, exist_ok=True)

        if EXPORT_FILE.exists():
            with open(EXPORT_FILE, "r", encoding="utf-8") as f:
                all_data = json.load(f)
        else:
            all_data = {}

        # subject 単位で上書き
        all_data[subjectNo] = export_data

        # -----------------------------
        # JSON 書き込み
        # -----------------------------
        with open(EXPORT_FILE, "w", encoding="utf-8") as f:
            json.dump(all_data, f, ensure_ascii=False, indent=2)

        self.stdout.write(
            self.style.SUCCESS(
                f"export 完了: {EXPORT_FILE} (subject={subjectNo})"
            )
        )
```



------





# **✅ このコードで保証されること**





✔ subject 単位で export

✔ **既存 JSON があっても壊さない（subject 部分のみ置換）**

✔ question_id 非依存（q_no 使用）

✔ exam_id 非依存

✔ 学生が A/B どちらかでも安全

✔ DB差異があっても import 可能な構造



------





# **🧪 実行例**



```
python manage.py export_subject_scores 1010401
```

結果：

```
exam2/data/export/examTFdata.json
```



------





# **🧭 次のステップ（確定）**





1️⃣ **この export を実行して JSON を目視確認**

2️⃣ OK なら

```
git add exam2/management/commands/export_subject_scores.py
git commit -m "Phase1.5 export subject scores (TF/hosei/adjust)"
```

3️⃣ **import_subject_scores.py 作成へ進む**



------



次は

👉 **import_subject_scores.py（完全コード）**

に進みますが、

その前に **JSON の中身を一度確認しますか？**



（problem_hash / q_no / students が期待通りか）









了解です。

では **Phase 1.5 の対になる処理**として、



> ✅ **import_subject_scores.py（完全コード）**



を提示します。

これは **先ほど提示した export_subject_scores.py と完全に対になる設計**です。



------





# **📄 import_subject_scores.py（完全版）**





**配置場所**

```
exam2/management/commands/import_subject_scores.py
```



------



```
from django.core.management.base import BaseCommand
from django.conf import settings
from django.db import transaction
from pathlib import Path
import json

from exam2.models import (
    Subject,
    Exam,
    Question,
    Student,
    StudentExam,
    ExamAdjust,
    StudentExamVersion,
)


IMPORT_FILE = Path(settings.BASE_DIR) / "exam2" / "data" / "export" / "examTFdata.json"


class Command(BaseCommand):
    help = "subject 単位で StudentExam(TF/hosei) と ExamAdjust を JSON から import する"

    def add_arguments(self, parser):
        parser.add_argument("subjectNo", type=str)
        parser.add_argument(
            "--dry-run",
            action="store_true",
            help="DB には書き込まず、内容確認のみ行う"
        )

    @transaction.atomic
    def handle(self, *args, **options):
        subjectNo = options["subjectNo"]
        dry_run = options["dry_run"]

        if not IMPORT_FILE.exists():
            self.stderr.write(self.style.ERROR("import 用 JSON が存在しません"))
            return

        with open(IMPORT_FILE, "r", encoding="utf-8") as f:
            all_data = json.load(f)

        if subjectNo not in all_data:
            self.stderr.write(self.style.ERROR("指定 subjectNo のデータがありません"))
            return

        data = all_data[subjectNo]

        subject = Subject.objects.filter(subjectNo=subjectNo).first()
        if not subject:
            self.stderr.write(self.style.ERROR("Subject が DB に存在しません"))
            return

        fsyear = data["fsyear"]
        term = data["term"]
        problem_hash = data.get("problem_hash")

        exams = Exam.objects.filter(
            subject=subject,
            fsyear=fsyear,
            term=term
        )

        if not exams.exists():
            self.stderr.write(self.style.ERROR("対象 Exam が存在しません"))
            return

        self.stdout.write(
            f"import subject={subjectNo}, fsyear={fsyear}, term={term}"
        )

        # --------------------------------
        # problem_hash チェック（警告のみ）
        # --------------------------------
        exam0 = exams.first()
        if exam0.problem_hash and problem_hash and exam0.problem_hash != problem_hash:
            self.stdout.write(
                self.style.WARNING(
                    "⚠ problem_hash が一致しません（処理は継続）"
                )
            )

        students_data = data["students"]

        update_count = 0

        for stdNo, stu_data in students_data.items():
            student = Student.objects.filter(stdNo=stdNo).first()
            if not student:
                self.stdout.write(
                    self.style.WARNING(f"学生未存在: {stdNo} → skip")
                )
                continue

            version = stu_data["version"]

            exam = exams.filter(version=version).first()
            if not exam:
                self.stdout.write(
                    self.style.WARNING(
                        f"Exam version={version} が存在しません → skip"
                    )
                )
                continue

            # --------------------------
            # StudentExamVersion 確認
            # --------------------------
            sev, _ = StudentExamVersion.objects.get_or_create(
                student=student,
                exam=exam
            )

            # --------------------------
            # StudentExam 更新
            # --------------------------
            for ans in stu_data["answers"]:
                q_no = ans["q_no"]

                question = Question.objects.filter(
                    exam=exam,
                    q_no=q_no
                ).first()

                if not question:
                    self.stdout.write(
                        self.style.WARNING(
                            f"question q_no={q_no} 不存在 → skip"
                        )
                    )
                    continue

                se, _ = StudentExam.objects.get_or_create(
                    student=student,
                    exam=exam,
                    question=question,
                    defaults={
                        "TF": ans["TF"],
                        "hosei": ans.get("hosei", 0)
                    }
                )

                if not dry_run:
                    se.TF = ans["TF"]
                    se.hosei = ans.get("hosei", 0)
                    se.save()

                update_count += 1

            # --------------------------
            # ExamAdjust 更新
            # --------------------------
            adjust_value = stu_data.get("adjust", 0)

            adj, created = ExamAdjust.objects.get_or_create(
                student=student,
                exam=exam,
                defaults={"adjust": adjust_value}
            )

            if not created and not dry_run:
                adj.adjust = adjust_value
                adj.save()

        if dry_run:
            self.stdout.write(
                self.style.WARNING(
                    f"[DRY-RUN] 更新予定件数: {update_count}"
                )
            )
            transaction.set_rollback(True)
        else:
            self.stdout.write(
                self.style.SUCCESS(
                    f"import 完了: 更新件数 {update_count}"
                )
            )
```



------





# **✅ この import の重要ポイント**







### **🔒 安全設計**





- question_id ❌ 不使用
- exam_id ❌ 不使用
- **q_no + version + subject** **で再構築**







### **🟡 hash の扱い**





- problem_hash は **警告のみ**
- 不一致でも止めない（Phase 1.5 方針）







### **🧪 dry-run 対応**



```
python manage.py import_subject_scores 1010401 --dry-run
```



------





# **🔁 export → import の完全対応関係**



| **項目**       | **export** | **import** |
| -------------- | ---------- | ---------- |
| subject 単位   | ✅          | ✅          |
| JSON 1ファイル | ✅          | ✅          |
| q_no 使用      | ✅          | ✅          |
| version 使用   | ✅          | ✅          |
| hash チェック  | 保存       | 警告       |
| dry-run        | –          | ✅          |



------





# **🧭 次のアクション**





1️⃣ **dry-run 実行**

2️⃣ JSON / ログ確認

3️⃣ 問題なければ本実行

4️⃣ **ここで git commit**

```
git add exam2/management/commands/import_subject_scores.py
git commit -m "Phase1.5 import subject scores (TF/hosei/adjust)"
```



------



次は

👉 **Phase 2（hash を使った自動判定 / import 制御）**

または

👉 **運用フロー（本番↔hotfix↔開発）整理**



どちらに進みますか？



了解です 👍

では **Phase 2：hash を使った自動判定 / import 制御** を

**「hash が違ったら import を中止する仕様」**として、**実装レベルで整理**します。



------





# **✅ Phase 2：hash を使った自動判定 / import 制御**







## **🎯 目的（Phase 1.5 との違い）**



| **フェーズ** | **hash の扱い**                           |
| ------------ | ----------------------------------------- |
| Phase 1.5    | hash 不一致 → **警告のみ・処理続行**      |
| **Phase 2**  | hash 不一致 → **import 中止（安全第一）** |

👉 **「別バージョンの問題に誤って採点データを流し込まない」**

これが Phase 2 の唯一かつ重要な目的です。



------





## **🧩 前提（ここまでで達成済み）**





- Exam に problem_hash が保存されている
- export JSON に problem_hash が入っている
- hash は **exam version 単位（A/B）ではなく exam セット単位**





👉 つまり

**Exam.problem_hash ≒ answers_xxxx.json の metainfo.hash**



------





## **🔁 Phase 2 の import 動作仕様（確定）**







### **1️⃣ import 開始時に hash を比較**



```
JSON.problem_hash  vs  DB.Exam.problem_hash
```



### **2️⃣ 判定ルール**



| **条件**              | **動作**                 |
| --------------------- | ------------------------ |
| DB 側に hash がない   | ❌ 中止（設計不整合）     |
| JSON 側に hash がない | ❌ 中止（古い export）    |
| hash が一致           | ✅ import 実行            |
| **hash が不一致**     | 🚨 **即中止（rollback）** |



------





## **🛑 import 中止時の振る舞い（重要）**





- **DB 変更なし**
- **transaction.atomic により自動 rollback**
- 明確なエラーメッセージを表示





例：

```
ERROR: problem_hash mismatch
DB   : 80acce50ab64dee048a4c084572e6652
JSON : 9f1bca77d0f4c8b4b62e1f9d2b9f0c12
```



------





## **🛠️ import_subject_scores.py の修正点（核心）**







### **🔴 Phase 1.5 のこの部分（警告のみ）**



```
if exam0.problem_hash and problem_hash and exam0.problem_hash != problem_hash:
    self.stdout.write(
        self.style.WARNING("⚠ problem_hash が一致しません（処理は継続）")
    )
```



------





### **🟢 Phase 2（**

### **中止仕様**

### **）に変更**



```
# --------------------------------
# problem_hash 厳密チェック（Phase 2）
# --------------------------------
if not exam0.problem_hash:
    self.stderr.write(
        self.style.ERROR("DB 側 Exam.problem_hash が未設定です")
    )
    transaction.set_rollback(True)
    return

if not problem_hash:
    self.stderr.write(
        self.style.ERROR("JSON 側 problem_hash が存在しません")
    )
    transaction.set_rollback(True)
    return

if exam0.problem_hash != problem_hash:
    self.stderr.write(
        self.style.ERROR(
            "problem_hash mismatch\n"
            f" DB   : {exam0.problem_hash}\n"
            f" JSON : {problem_hash}"
        )
    )
    transaction.set_rollback(True)
    return
```

👉 **この return があることで、以降の import 処理は一切走りません**



------





## **🔒 なぜここで止めていいのか（設計的理由）**





- 問題が違う試験に

  

  - TF

  - hosei

  - adjust

    を入れることは **100% 事故**

  

- 「少し違うけどたぶん同じ」は **人間の錯覚**

- hash は

  **「人間の判断を置き換えるための安全装置」**





👉 Phase 2 では

**人間の確認ステップをコードに昇格**させています。



------





## **🧪 テスト方法（Phase 2）**







### **正常系**



```
python manage.py import_subject_scores 1010401
```



### **異常系（hash 不一致）**





1. JSON をコピーして problem_hash を 1 文字変える
2. 実行



```
python manage.py import_subject_scores 1010401
```

期待結果：



- import されない
- DB に変更なし
- エラーメッセージ表示





------





## **🗂️ git 運用（強くおすすめ）**







### **コミット単位**



```
Phase2: strict problem_hash validation on import
git add exam2/management/commands/import_subject_scores.py
git commit -m "Phase2: abort import when problem_hash mismatch"
```

👉 **この commit は「安全装置を入れた」節目です**



------





## **🧭 次に進める Phase（予告）**





Phase 2 完了後に選べます：





### **Phase 3（発展）**





- hash 不一致時に

  

  - 差分表示
  - 強制 import オプション --force

  







### **Phase 4（完成形）**





- UI に hash 表示
- export / import 履歴管理
- 本番・hotfix・開発の完全分離運用





------



まずは

👉 **Phase 2 を実装 → テスト → commit**

ここまで一気にやりましょう。



実装後、次は

**「–force を許すか？」**

**「import を UI からできるようにするか？」**



どこまで行くか、一緒に決めましょう。