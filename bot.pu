import os
import secrets
import sqlite3
from contextlib import contextmanager
from datetime import date

from telegram import InlineKeyboardButton as B, InlineKeyboardMarkup as M, Update
from telegram.ext import Application, CallbackQueryHandler, CommandHandler, ContextTypes, MessageHandler, filters

TOKEN = os.environ['BOT_TOKEN']
TEACHER_CODE = os.environ['TEACHER_CODE']
DB_PATH = os.environ.get('DB_PATH', '/data/bilim.db')
os.makedirs(os.path.dirname(os.path.abspath(DB_PATH)), exist_ok=True)

QUESTIONS = {
    1: [
        ('Matematika', '3 + 2 = ?', ['4', '5', '6'], 1, 'Uchga ikki qo‘shilsa besh bo‘ladi.'),
        ('Matematika', '7 - 4 = ?', ['2', '3', '4'], 1, 'Yettidan to‘rtni ayirsak uch qoladi.'),
        ('Harflar', '«Olma» so‘zi qaysi harf bilan boshlanadi?', ['O', 'A', 'M'], 0, 'Olma so‘zi O harfi bilan boshlanadi.'),
        ('Harflar', '«Kitob» so‘zida nechta harf bor?', ['4', '5', '6'], 1, 'K-i-t-o-b — beshta harf.'),
        ('Mantiq', 'Qaysi biri meva?', ['Sabzi', 'Olma', 'Kartoshka'], 1, 'Olma — meva.'),
        ('Olam', 'Quyosh qachon chiqadi?', ['Ertalab', 'Kechasi', 'Yarim tunda'], 0, 'Quyosh ertalab chiqadi.'),
    ],
    2: [
        ('Matematika', '12 + 7 = ?', ['18', '19', '20'], 1, '12 ga 7 qo‘shsak 19 bo‘ladi.'),
        ('Matematika', '20 - 8 = ?', ['12', '13', '14'], 0, '20 dan 8 ni ayirsak 12 qoladi.'),
        ('Harflar', 'Qaysi so‘z uch bo‘g‘inli?', ['Olma', 'Kapalak', 'Gul'], 1, 'Ka-pa-lak — uch bo‘g‘in.'),
        ('Harflar', '«Bolalar» so‘zida nechta harf bor?', ['6', '7', '8'], 1, 'B-o-l-a-l-a-r — yettita harf.'),
        ('Mantiq', 'Dushanbadan keyin qaysi kun keladi?', ['Seshanba', 'Yakshanba', 'Juma'], 0, 'Dushanbadan keyin seshanba keladi.'),
        ('Olam', 'O‘simlik o‘sishi uchun nima kerak?', ['Faqat tosh', 'Suv va yorug‘lik', 'Faqat qog‘oz'], 1, 'O‘simlikka suv va yorug‘lik kerak.'),
    ],
}

@contextmanager
def db():
    con = sqlite3.connect(DB_PATH, timeout=20)
    con.row_factory = sqlite3.Row
    try:
        yield con
        con.commit()
    finally:
        con.close()

def init_db():
    with db() as c:
        c.execute('PRAGMA journal_mode=WAL')
        c.executescript('''
        CREATE TABLE IF NOT EXISTS teachers(uid INTEGER PRIMARY KEY);
        CREATE TABLE IF NOT EXISTS students(id INTEGER PRIMARY KEY, teacher INTEGER NOT NULL,
          name TEXT NOT NULL, grade INTEGER NOT NULL, code TEXT UNIQUE NOT NULL, points INTEGER NOT NULL DEFAULT 0,
          UNIQUE(teacher, name));
        CREATE TABLE IF NOT EXISTS parents(uid INTEGER NOT NULL, student INTEGER NOT NULL,
          PRIMARY KEY(uid, student));
        CREATE TABLE IF NOT EXISTS teams(id INTEGER PRIMARY KEY, teacher INTEGER NOT NULL,
          name TEXT NOT NULL, grade INTEGER NOT NULL, points INTEGER NOT NULL DEFAULT 0);
        CREATE TABLE IF NOT EXISTS awards(kind TEXT NOT NULL, object_id INTEGER NOT NULL, uid INTEGER NOT NULL,
          day TEXT NOT NULL, question INTEGER NOT NULL, PRIMARY KEY(kind, object_id, uid, day, question));
        ''')

def keyboard(rows):
    return M([[B(label, callback_data=value) for label, value in row] for row in rows])

def is_teacher(uid):
    with db() as c:
        return c.execute('SELECT 1 FROM teachers WHERE uid=?', (uid,)).fetchone() is not None

def my_students(uid):
    with db() as c:
        return c.execute('SELECT s.* FROM students s JOIN parents p ON p.student=s.id WHERE p.uid=? ORDER BY s.name', (uid,)).fetchall()

def menu(uid):
    rows = []
    if is_teacher(uid):
        rows += [[('➕ O‘quvchi qo‘shish', 'addstudent'), ('👧 O‘quvchilar', 'students')],
                 [('🎯 Yakka savol', 'single'), ('👥 Jamoa o‘yini', 'teams')],
                 [('📊 Natijalar', 'results')]]
    rows += [[('🏠 Uyda mashq', 'home'), ('🔗 Bolani ulash', 'link')]]
    return keyboard(rows)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data.clear()
    await update.effective_message.reply_text(
        '🌟 Bilim sayohatiga xush kelibsiz!\n\nUstoz bo‘lsangiz /ustoz buyrug‘ini yuboring. Ota-ona bo‘lsangiz «Bolani ulash»ni bosing.',
        reply_markup=menu(update.effective_user.id))

async def teacher(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if is_teacher(update.effective_user.id):
        await update.message.reply_text('Ustoz menyusi:', reply_markup=menu(update.effective_user.id))
    else:
        context.user_data['input'] = 'teacher_code'
        await update.message.reply_text('Ustoz maxfiy kodini kiriting. Kod faqat bot sozlamasida saqlansin.')

async def show_menu(q, uid):
    await q.edit_message_text('🌟 Bilim sayohati — bo‘limni tanlang:', reply_markup=menu(uid))

async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    uid = q.from_user.id
    data = q.data
    if data == 'menu':
        context.user_data.pop('quiz', None)
        await show_menu(q, uid)
        return
    if data == 'link':
        context.user_data['input'] = 'link'
        await q.edit_message_text('Ustoz bergan ulanish kodini yozing.', reply_markup=keyboard([[('⬅️ Menyu', 'menu')]]))
        return
    if data == 'home':
        children = my_students(uid)
        if not children:
            await q.edit_message_text('Avval bolani ustoz bergan kod bilan ulang.', reply_markup=keyboard([[('🔗 Bolani ulash', 'link'), ('⬅️ Menyu', 'menu')]]))
        else:
            await q.edit_message_text('Uyda mashq qiladigan bolani tanlang:', reply_markup=keyboard(
                [[(f'{s["name"]} ({s["grade"]}-sinf)', f'homechild:{s["id"]}')] for s in children] + [[('⬅️ Menyu', 'menu')]]))
        return
    if data.startswith('homechild:'):
        sid = int(data.split(':')[1])
        if not any(s['id'] == sid for s in my_students(uid)):
            await q.edit_message_text('Bu bola sizga ulanmagan.')
            return
        await choose_subject(q, context, 'student', sid, uid)
        return
    if data in ('addstudent', 'students', 'single', 'teams', 'results') or data.startswith(('student:', 'team:', 'newteam:', 'subject:', 'answer:', 'reveal:', 'teamaward:', 'grade:', 'next:')):
        if not is_teacher(uid) and not data.startswith(('subject:', 'answer:')):
            await q.edit_message_text('Bu bo‘lim faqat ustoz uchun.')
            return
    if data == 'addstudent':
        context.user_data['input'] = 'student'
        await q.edit_message_text('Bolaning ismi va sinfini shunday yozing:\nAli 1\n\nBir xil ismli bolalar bo‘lsa, familiyasini ham kiriting.')
    elif data == 'students':
        with db() as c:
            rows = c.execute('SELECT * FROM students WHERE teacher=? ORDER BY grade,name', (uid,)).fetchall()
        msg = '\n'.join(f'{s["name"]} — {s["grade"]}-sinf, {s["points"]} ball, kod: {s["code"]}' for s in rows)
        await q.edit_message_text('👧 O‘quvchilar:\n' + (msg or 'Hozircha yo‘q.'), reply_markup=keyboard([[('⬅️ Menyu', 'menu')]]))
    elif data == 'results':
        with db() as c:
            rows = c.execute('SELECT name,grade,points FROM students WHERE teacher=? ORDER BY grade,points DESC', (uid,)).fetchall()
            teams = c.execute('SELECT name,points FROM teams WHERE teacher=? ORDER BY points DESC', (uid,)).fetchall()
        a = '\n'.join(f'{s["grade"]}-sinf: {s["name"]} — {s["points"]} ball' for s in rows) or 'Hozircha yo‘q.'
        b = '\n'.join(f'{t["name"]} — {t["points"]} ball' for t in teams) or 'Hozircha yo‘q.'
        await q.edit_message_text('📊 Shaxsiy ballar:\n' + a + '\n\n👥 Jamoa ballari:\n' + b, reply_markup=keyboard([[('⬅️ Menyu', 'menu')]]))
    elif data == 'single':
        with db() as c:
            rows = c.execute('SELECT id,name,grade FROM students WHERE teacher=? ORDER BY name', (uid,)).fetchall()
        await q.edit_message_text('Javob beradigan bolani tanlang:', reply_markup=keyboard(
            [[(f'{s["name"]} ({s["grade"]}-sinf)', f'student:{s["id"]}')] for s in rows] + [[('⬅️ Menyu', 'menu')]]))
    elif data.startswith('student:'):
        sid = int(data.split(':')[1])
        with db() as c:
            s = c.execute('SELECT 1 FROM students WHERE id=? AND teacher=?', (sid, uid)).fetchone()
        if s:
            await choose_subject(q, context, 'student', sid, uid)
    elif data == 'teams':
        await q.edit_message_text('Jamoalar uchun sinfni tanlang:', reply_markup=keyboard([[('1-sinf', 'grade:1'), ('2-sinf', 'grade:2')], [('⬅️ Menyu', 'menu')]]))
    elif data.startswith('grade:'):
        grade = int(data.split(':')[1])
        with db() as c:
            rows = c.execute('SELECT id,name,points FROM teams WHERE teacher=? AND grade=? ORDER BY id', (uid, grade)).fetchall()
        await q.edit_message_text(f'{grade}-sinf jamoalari:', reply_markup=keyboard(
            [[(f'{t["name"]} — {t["points"]} ball', f'team:{t["id"]}')] for t in rows] +
            [[('➕ Jamoa qo‘shish', f'newteam:{grade}')], [('⬅️ Menyu', 'menu')]]))
    elif data.startswith('newteam:'):
        context.user_data['input'] = ('team', int(data.split(':')[1]))
        await q.edit_message_text('Jamoaning nomini yozing. Masalan: Zukkolar')
    elif data.startswith('team:'):
        tid = int(data.split(':')[1])
        with db() as c:
            row = c.execute('SELECT 1 FROM teams WHERE id=? AND teacher=?', (tid, uid)).fetchone()
        if row:
            await choose_subject(q, context, 'team', tid, uid)
    elif data.startswith('next:'):
        quiz = context.user_data.get('quiz')
        if quiz and quiz['owner'] == uid:
            await show_question(q, quiz)
    elif data.startswith('subject:'):
        quiz = context.user_data.get('quiz')
        if quiz and quiz['owner'] == uid:
            quiz['subject'] = data.split(':', 1)[1]
            quiz['index'] = 0
            await show_question(q, quiz)
    elif data.startswith('answer:'):
        quiz = context.user_data.get('quiz')
        if not quiz or quiz['owner'] != uid or quiz['kind'] != 'student':
            await q.edit_message_text('Mashq tugagan. /start ni bosing.')
            return
        qid, selected = map(int, data.split(':')[1:])
        current = quiz_question(quiz)
        if current is None or qid != quiz['index']:
            await q.answer('Bu savol allaqachon yakunlangan.', show_alert=True)
            return
        original_id = selected_questions(quiz)[quiz['index']][0]
        correct = current[3]
        if selected == correct:
            awarded = award('student', quiz['id'], uid, quiz['grade'], original_id)
            result = '✅ To‘g‘ri! ' + ('+10 ball' if awarded else 'Bu savol uchun bugun ball berilgan.')
        else:
            result = '💡 To‘g‘ri javob: ' + current[2][correct]
        quiz['index'] += 1
        await q.edit_message_text(result + '\n' + current[4], reply_markup=keyboard([[('➡️ Keyingi savol', 'next:1')], [('⬅️ Menyu', 'menu')]]))
    elif data.startswith('reveal:'):
        quiz = context.user_data.get('quiz')
        if not quiz or quiz['owner'] != uid or quiz['kind'] != 'team':
            return
        current = quiz_question(quiz)
        if current:
            await q.edit_message_text(f'✅ Javob: {current[2][current[3]]}\n{current[4]}\n\nJamoa to‘g‘ri javob berdimi?', reply_markup=keyboard(
                [[('✅ +10 ball', f'teamaward:{quiz["index"]}:1'), ('❌ 0 ball', f'teamaward:{quiz["index"]}:0')]]))
    elif data.startswith('teamaward:'):
        quiz = context.user_data.get('quiz')
        if not quiz or quiz['owner'] != uid or quiz['kind'] != 'team':
            return
        _, index, ok = data.split(':')
        if int(index) != quiz['index']:
            return
        original_id = selected_questions(quiz)[quiz['index']][0]
        awarded = award('team', quiz['id'], uid, quiz['grade'], original_id) if ok == '1' else False
        quiz['index'] += 1
        await show_question(q, quiz, prefix=('✅ +10 ball\n\n' if awarded else '➡️ Keyingi savol\n\n'))

async def choose_subject(q, context, kind, object_id, uid):
    with db() as c:
        table = 'students' if kind == 'student' else 'teams'
        row = c.execute(f'SELECT grade,name FROM {table} WHERE id=?', (object_id,)).fetchone()
    if not row:
        return
    context.user_data['quiz'] = {'kind': kind, 'id': object_id, 'grade': row['grade'], 'owner': uid, 'index': 0, 'subject': 'Barchasi'}
    await q.edit_message_text(f'{row["name"]}: mavzuni tanlang.', reply_markup=keyboard(
        [[('🌟 Barchasi', 'subject:Barchasi')], [('🔢 Matematika', 'subject:Matematika'), ('🔤 Harflar', 'subject:Harflar')],
         [('🧩 Mantiq', 'subject:Mantiq'), ('🌿 Olam', 'subject:Olam')], [('⬅️ Menyu', 'menu')]]))

def selected_questions(quiz):
    questions = QUESTIONS[quiz['grade']]
    return [(i, question) for i, question in enumerate(questions) if quiz['subject'] == 'Barchasi' or question[0] == quiz['subject']]

def quiz_question(quiz):
    items = selected_questions(quiz)
    return items[quiz['index']][1] if quiz['index'] < len(items) else None

async def show_question(q, quiz, prefix=''):
    items = selected_questions(quiz)
    if quiz['index'] >= len(items):
        await q.edit_message_text(prefix + '🏁 Savollar tugadi!', reply_markup=keyboard([[('⬅️ Menyu', 'menu')]]))
        return
    _, question = items[quiz['index']]
    number = quiz['index'] + 1
    if quiz['kind'] == 'team':
        rows = [[('🔍 Javobni ko‘rsatish', f'reveal:{number}')]]
        text = f'{prefix}👥 {number}/{len(items)}. {question[1]}\n\n' + '\n'.join(f'{i+1}. {a}' for i, a in enumerate(question[2]))
    else:
        rows = [[(a, f'answer:{quiz["index"]}:{i}')] for i, a in enumerate(question[2])]
        text = f'{prefix}🌟 {number}/{len(items)}. {question[1]}'
    rows.append([('⬅️ Menyu', 'menu')])
    await q.edit_message_text(text, reply_markup=keyboard(rows))

def award(kind, object_id, uid, grade, qid):
    with db() as c:
        try:
            c.execute('INSERT INTO awards VALUES (?,?,?,?,?)', (kind, object_id, uid, date.today().isoformat(), qid))
        except sqlite3.IntegrityError:
            return False
        c.execute(f'UPDATE {"students" if kind == "student" else "teams"} SET points=points+10 WHERE id=? AND grade=?', (object_id, grade))
    return True

async def message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    value = update.message.text.strip()
    uid = update.effective_user.id
    state = context.user_data.pop('input', None)
    if state == 'teacher_code':
        if secrets.compare_digest(value, TEACHER_CODE):
            with db() as c:
                c.execute('INSERT OR IGNORE INTO teachers VALUES (?)', (uid,))
            await update.message.reply_text('✅ Ustoz sifatida ulandingiz.', reply_markup=menu(uid))
        else:
            await update.message.reply_text('Kod xato. Qayta urinish uchun /ustoz ni bosing.')
    elif state == 'link':
        with db() as c:
            s = c.execute('SELECT id,name FROM students WHERE code=?', (value.upper(),)).fetchone()
            if s:
                c.execute('INSERT OR IGNORE INTO parents VALUES (?,?)', (uid, s['id']))
        if s:
            await update.message.reply_text(f'✅ {s["name"]} ulandi.', reply_markup=menu(uid))
        else:
            await update.message.reply_text('Bunday kod topilmadi. Ustozdan kodni tekshirib oling.')
    elif state == 'student' and is_teacher(uid):
        try:
            name, grade_text = value.rsplit(' ', 1)
            grade = int(grade_text)
            if grade not in (1, 2) or not (1 <= len(name) <= 80):
                raise ValueError
        except ValueError:
            await update.message.reply_text('Masalan: Ali Karimov 1\nQayta boshlash uchun «O‘quvchi qo‘shish»ni bosing.', reply_markup=menu(uid))
            return
        code = secrets.token_hex(4).upper()
        try:
            with db() as c:
                c.execute('INSERT INTO students(teacher,name,grade,code) VALUES (?,?,?,?)', (uid, name, grade, code))
        except sqlite3.IntegrityError:
            await update.message.reply_text('Shu ismli bola allaqachon bor. Familiyasini ham qo‘shing.', reply_markup=menu(uid))
            return
        await update.message.reply_text(f'✅ {name}, {grade}-sinf qo‘shildi.\nOta-onaga faqat shu kodni bering: {code}', reply_markup=menu(uid))
    elif isinstance(state, tuple) and state[0] == 'team' and is_teacher(uid):
        grade = state[1]
        if not 1 <= len(value) <= 50:
            await update.message.reply_text('Jamoa nomi 1–50 belgidan iborat bo‘lsin.')
            return
        with db() as c:
            c.execute('INSERT INTO teams(teacher,name,grade) VALUES (?,?,?)', (uid, value, grade))
        await update.message.reply_text(f'✅ {value} jamoasi qo‘shildi.', reply_markup=menu(uid))
    else:
        await update.message.reply_text('Menyuni ochish uchun /start ni bosing.')

async def error(update, context):
    import logging
    logging.exception('Bot xatosi', exc_info=context.error)

if __name__ == '__main__':
    if not TEACHER_CODE or len(TEACHER_CODE) < 8:
        raise RuntimeError('TEACHER_CODE kamida 8 belgidan iborat bo‘lsin')
    init_db()
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler('start', start))
    app.add_handler(CommandHandler('ustoz', teacher))
    app.add_handler(CallbackQueryHandler(callback))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, message))
    app.add_error_handler(error)
    app.run_polling(drop_pending_updates=True)
