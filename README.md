# python-project-
Python code
import asyncio
import os
import random

from openpyxl import load_workbook
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import CommandStart, Command
from aiogram.utils.keyboard import InlineKeyboardBuilder

# Tokeni kodun içinə yazma: terminalda  set BOT_TOKEN=...  (Windows)  /  export BOT_TOKEN=... (Linux/Mac)
BOT_TOKEN = os.getenv("BOT_TOKEN", "8896058854:AAG0iV5fU2LlPNOFHaQdJ_iysF2GjrzWqqI ")
EXCEL_FILE = "questions.xlsx"  # Google Sheets-dən "Download as .xlsx" ilə yüklənən fayl

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()


# ---------- 1. Excel-dən sualların oxunması ----------
def load_questions(path=EXCEL_FILE):
    """Birinci vərəqi oxuyur. Başlıqlar: İd, Sinif, Sınaq, Sual, A, B, C, D, Düzgün variant, Həlli"""
    ws = load_workbook(path, data_only=True).worksheets[0]
    rows = ws.iter_rows(values_only=True)
    header = [str(h).strip() if h else "" for h in next(rows)]
    questions = []
    for row in rows:
        r = dict(zip(header, row))
        if not r.get("Sual"):
            continue
        questions.append({
            "id": str(r.get("İd") or ""),
            "grade": str(r.get("Sinif") or "").strip(),
            "test": str(r.get("Sınaq") or "").strip(),
            "text": str(r["Sual"]),
            "options": {k: str(r.get(k) if r.get(k) is not None else "") for k in "ABCD"},
            "correct": str(r.get("Düzgün variant") or "").strip().upper(),
            "solution": str(r.get("Həlli") or "Həll yoxdur."),
        })
    return questions


QUESTIONS = load_questions()


def reload_questions():
    """Faylı yenidən oxuyur. Xəta olsa köhnə suallar qalır."""
    global QUESTIONS
    try:
        QUESTIONS = load_questions()
        return True
    except Exception as e:
        print("Excel oxunmadı:", e)
        return False
sessions = {}  # user_id -> {"qs": [...], "i": 0, "score": 0, "answered": False}


def tests_for_grade(grade):
    seen = []
    for q in QUESTIONS:
        if q["grade"] == grade and q["test"] not in seen:
            seen.append(q["test"])
    return seen


# ---------- 2. Menyu (sənin kodun) ----------
def main_menu():
    b = InlineKeyboardBuilder()
    b.button(text="Math", callback_data="Math1")
    b.button(text="Help", callback_data="Help1")
    b.adjust(2)
    return b.as_markup()


@dp.message(CommandStart())
async def cmd_start(message: types.Message):
    await message.answer("Choose", reply_markup=main_menu())


@dp.message(Command("reload"))
async def cmd_reload(message: types.Message):
    ok = reload_questions()
    if ok:
        await message.answer(f"✅ Suallar yeniləndi. Cəmi: {len(QUESTIONS)} sual.")
    else:
        await message.answer("❌ Fayl oxunmadı. Köhnə suallar saxlanıldı.")


@dp.callback_query(F.data == "Math1")
async def math_menu(callback: types.CallbackQuery):
    b = InlineKeyboardBuilder()
    b.button(text="9-th grade", callback_data="grade:9")
    b.button(text="11-th grade", callback_data="grade:11")
    b.button(text="🔙 Back", callback_data="back_to_start")
    b.adjust(2, 1)
    await callback.message.edit_text("You chose MATH.", reply_markup=b.as_markup())
    await callback.answer()


@dp.callback_query(F.data == "Help1")
async def help_menu(callback: types.CallbackQuery):
    b = InlineKeyboardBuilder()
    b.button(text="🔙 Back", callback_data="back_to_start")
    await callback.message.edit_text(
        "This bot was created for practicing DIM exam.", reply_markup=b.as_markup()
    )
    await callback.answer()


@dp.callback_query(F.data == "back_to_start")
async def back_to_start(callback: types.CallbackQuery):
    await callback.message.edit_text("Choose.", reply_markup=main_menu())
    await callback.answer()


# ---------- 3. Sinif -> Sınaq seçimi ----------
@dp.callback_query(F.data.startswith("grade:"))
async def choose_grade(callback: types.CallbackQuery):
    grade = callback.data.split(":")[1]
    reload_questions()  # fayldakı yeni suallar avtomatik görünsün
    tests = tests_for_grade(grade)
    b = InlineKeyboardBuilder()
    for idx, t in enumerate(tests):
        b.button(text=t, callback_data=f"test:{grade}:{idx}")
    b.button(text="🔙 Back", callback_data="Math1")
    b.adjust(2)
    if not tests:
        await callback.message.edit_text(f"{grade}-ci sinif üçün sual yoxdur.", reply_markup=b.as_markup())
    else:
        await callback.message.edit_text(f"{grade}-ci sinif. Sınaq seçin:", reply_markup=b.as_markup())
    await callback.answer()


# ---------- 4. Sınağın başlaması ----------
@dp.callback_query(F.data.startswith("test:"))
async def start_test(callback: types.CallbackQuery):
    _, grade, idx = callback.data.split(":")
    test_name = tests_for_grade(grade)[int(idx)]
    qs = [q for q in QUESTIONS if q["grade"] == grade and q["test"] == test_name]
    # random.shuffle(qs)  # suallar qarışıq olsun istəsən, bu sətrin əvvəlindəki # işarəsini sil
    sessions[callback.from_user.id] = {"qs": qs, "i": 0, "score": 0, "answered": False}
    await send_question(callback)


async def send_question(callback: types.CallbackQuery):
    s = sessions[callback.from_user.id]
    q = s["qs"][s["i"]]
    s["answered"] = False
    text = (
        f"Sual {s['i'] + 1}/{len(s['qs'])}\n\n{q['text']}\n\n"
        + "\n".join(f"{k}) {v}" for k, v in q["options"].items())
    )
    b = InlineKeyboardBuilder()
    for k in "ABCD":
        b.button(text=k, callback_data=f"ans:{k}")
    b.button(text="⛔ Dayandır", callback_data="stop")
    b.adjust(4, 1)
    await callback.message.edit_text(text, reply_markup=b.as_markup())
    await callback.answer()


# ---------- 5. Cavabın yoxlanması ----------
@dp.callback_query(F.data.startswith("ans:"))
async def check_answer(callback: types.CallbackQuery):
    s = sessions.get(callback.from_user.id)
    if not s:
        await callback.answer("Sınaq başlamayıb. /start yazın.", show_alert=True)
        return
    if s["answered"]:
        await callback.answer("Artıq cavab vermisiniz.")
        return
    s["answered"] = True

    q = s["qs"][s["i"]]
    chosen = callback.data.split(":")[1]
    if chosen == q["correct"]:
        s["score"] += 1
        verdict = "✅ Düzgündür!"
    else:
        verdict = f"❌ Səhvdir. Sizin cavab: {chosen}"

    last = s["i"] + 1 >= len(s["qs"])
    b = InlineKeyboardBuilder()
    b.button(text="🏁 Nəticə" if last else "➡️ Növbəti", callback_data="next")

    text = (
        f"{q['text']}\n\n{verdict}\n"
        f"Düzgün variant: {q['correct']}) {q['options'].get(q['correct'], '')}\n\n"
        f"Həlli:\n{q['solution']}"
    )
    await callback.message.edit_text(text, reply_markup=b.as_markup())
    await callback.answer()


# ---------- 6. Növbəti sual / nəticə ----------
@dp.callback_query(F.data == "next")
async def next_question(callback: types.CallbackQuery):
    s = sessions.get(callback.from_user.id)
    if not s:
        await callback.answer("Sınaq başlamayıb. /start yazın.", show_alert=True)
        return
    s["i"] += 1
    if s["i"] >= len(s["qs"]):
        await show_result(callback)
    else:
        await send_question(callback)


@dp.callback_query(F.data == "stop")
async def stop_test(callback: types.CallbackQuery):
    await show_result(callback)


async def show_result(callback: types.CallbackQuery):
    s = sessions.pop(callback.from_user.id, None)
    if s:
        answered = min(s["i"] + (1 if s["answered"] else 0), len(s["qs"]))
        text = f"Sınaq bitdi.\nNəticə: {s['score']}/{len(s['qs'])} düzgün cavab."
    else:
        text = "Sınaq bitdi."
    b = InlineKeyboardBuilder()
    b.button(text="🏠 Əsas menyu", callback_data="back_to_start")
    await callback.message.edit_text(text, reply_markup=b.as_markup())
    await callback.answer()


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
import asyncio
import os
import random

from openpyxl import load_workbook
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import CommandStart, Command
from aiogram.utils.keyboard import InlineKeyboardBuilder

# Tokeni kodun içinə yazma: terminalda  set BOT_TOKEN=...  (Windows)  /  export BOT_TOKEN=... (Linux/Mac)
BOT_TOKEN = os.getenv("BOT_TOKEN", "")
EXCEL_FILE = "questions.xlsx"  # Google Sheets-dən "Download as .xlsx" ilə yüklənən fayl

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()


# ---------- 1. Excel-dən sualların oxunması ----------
def load_questions(path=EXCEL_FILE):
    """Birinci vərəqi oxuyur. Başlıqlar: İd, Sinif, Sınaq, Sual, A, B, C, D, Düzgün variant, Həlli"""
    ws = load_workbook(path, data_only=True).worksheets[0]
    rows = ws.iter_rows(values_only=True)
    header = [str(h).strip() if h else "" for h in next(rows)]
    questions = []
    for row in rows:
        r = dict(zip(header, row))
        if not r.get("Sual"):
            continue
        questions.append({
            "id": str(r.get("İd") or ""),
            "grade": str(r.get("Sinif") or "").strip(),
            "test": str(r.get("Sınaq") or "").strip(),
            "text": str(r["Sual"]),
            "options": {k: str(r.get(k) if r.get(k) is not None else "") for k in "ABCD"},
            "correct": str(r.get("Düzgün variant") or "").strip().upper(),
            "solution": str(r.get("Həlli") or "Həll yoxdur."),
        })
    return questions


QUESTIONS = load_questions()


def reload_questions():
    """Faylı yenidən oxuyur. Xəta olsa köhnə suallar qalır."""
    global QUESTIONS
    try:
        QUESTIONS = load_questions()
        return True
    except Exception as e:
        print("Excel oxunmadı:", e)
        return False
sessions = {}  # user_id -> {"qs": [...], "i": 0, "score": 0, "answered": False}


def tests_for_grade(grade):
    seen = []
    for q in QUESTIONS:
        if q["grade"] == grade and q["test"] not in seen:
            seen.append(q["test"])
    return seen


# ---------- 2. Menyu (sənin kodun) ----------
def main_menu():
    b = InlineKeyboardBuilder()
    b.button(text="Math", callback_data="Math1")
    b.button(text="Help", callback_data="Help1")
    b.adjust(2)
    return b.as_markup()


@dp.message(CommandStart())
async def cmd_start(message: types.Message):
    await message.answer("Choose", reply_markup=main_menu())


@dp.message(Command("reload"))
async def cmd_reload(message: types.Message):
    ok = reload_questions()
    if ok:
        await message.answer(f"✅ Suallar yeniləndi. Cəmi: {len(QUESTIONS)} sual.")
    else:
        await message.answer("❌ Fayl oxunmadı. Köhnə suallar saxlanıldı.")


@dp.callback_query(F.data == "Math1")
async def math_menu(callback: types.CallbackQuery):
    b = InlineKeyboardBuilder()
    b.button(text="9-th grade", callback_data="grade:9")
    b.button(text="11-th grade", callback_data="grade:11")
    b.button(text="🔙 Back", callback_data="back_to_start")
    b.adjust(2, 1)
    await callback.message.edit_text("You chose MATH.", reply_markup=b.as_markup())
    await callback.answer()


@dp.callback_query(F.data == "Help1")
async def help_menu(callback: types.CallbackQuery):
    b = InlineKeyboardBuilder()
    b.button(text="🔙 Back", callback_data="back_to_start")
    await callback.message.edit_text(
        "This bot was created for practicing DIM exam.", reply_markup=b.as_markup()
    )
    await callback.answer()


@dp.callback_query(F.data == "back_to_start")
async def back_to_start(callback: types.CallbackQuery):
    await callback.message.edit_text("Choose.", reply_markup=main_menu())
    await callback.answer()


# ---------- 3. Sinif -> Sınaq seçimi ----------
@dp.callback_query(F.data.startswith("grade:"))
async def choose_grade(callback: types.CallbackQuery):
    grade = callback.data.split(":")[1]
    reload_questions()  # fayldakı yeni suallar avtomatik görünsün
    tests = tests_for_grade(grade)
    b = InlineKeyboardBuilder()
    for idx, t in enumerate(tests):
        b.button(text=t, callback_data=f"test:{grade}:{idx}")
    b.button(text="🔙 Back", callback_data="Math1")
    b.adjust(2)
    if not tests:
        await callback.message.edit_text(f"{grade}-ci sinif üçün sual yoxdur.", reply_markup=b.as_markup())
    else:
        await callback.message.edit_text(f"{grade}-ci sinif. Sınaq seçin:", reply_markup=b.as_markup())
    await callback.answer()


# ---------- 4. Sınağın başlaması ----------
@dp.callback_query(F.data.startswith("test:"))
async def start_test(callback: types.CallbackQuery):
    _, grade, idx = callback.data.split(":")
    test_name = tests_for_grade(grade)[int(idx)]
    qs = [q for q in QUESTIONS if q["grade"] == grade and q["test"] == test_name]
    # random.shuffle(qs)  # suallar qarışıq olsun istəsən, bu sətrin əvvəlindəki # işarəsini sil
    sessions[callback.from_user.id] = {"qs": qs, "i": 0, "score": 0, "answered": False}
    await send_question(callback)


async def send_question(callback: types.CallbackQuery):
    s = sessions[callback.from_user.id]
    q = s["qs"][s["i"]]
    s["answered"] = False
    text = (
        f"Sual {s['i'] + 1}/{len(s['qs'])}\n\n{q['text']}\n\n"
        + "\n".join(f"{k}) {v}" for k, v in q["options"].items())
    )
    b = InlineKeyboardBuilder()
    for k in "ABCD":
        b.button(text=k, callback_data=f"ans:{k}")
    b.button(text="⛔ Dayandır", callback_data="stop")
    b.adjust(4, 1)
    await callback.message.edit_text(text, reply_markup=b.as_markup())
    await callback.answer()


# ---------- 5. Cavabın yoxlanması ----------
@dp.callback_query(F.data.startswith("ans:"))
async def check_answer(callback: types.CallbackQuery):
    s = sessions.get(callback.from_user.id)
    if not s:
        await callback.answer("Sınaq başlamayıb. /start yazın.", show_alert=True)
        return
    if s["answered"]:
        await callback.answer("Artıq cavab vermisiniz.")
        return
    s["answered"] = True

    q = s["qs"][s["i"]]
    chosen = callback.data.split(":")[1]
    if chosen == q["correct"]:
        s["score"] += 1
        verdict = "✅ Düzgündür!"
    else:
        verdict = f"❌ Səhvdir. Sizin cavab: {chosen}"

    last = s["i"] + 1 >= len(s["qs"])
    b = InlineKeyboardBuilder()
    b.button(text="🏁 Nəticə" if last else "➡️ Növbəti", callback_data="next")

    text = (
        f"{q['text']}\n\n{verdict}\n"
        f"Düzgün variant: {q['correct']}) {q['options'].get(q['correct'], '')}\n\n"
        f"Həlli:\n{q['solution']}"
    )
    await callback.message.edit_text(text, reply_markup=b.as_markup())
    await callback.answer()


# ---------- 6. Növbəti sual / nəticə ----------
@dp.callback_query(F.data == "next")
async def next_question(callback: types.CallbackQuery):
    s = sessions.get(callback.from_user.id)
    if not s:
        await callback.answer("Sınaq başlamayıb. /start yazın.", show_alert=True)
        return
    s["i"] += 1
    if s["i"] >= len(s["qs"]):
        await show_result(callback)
    else:
        await send_question(callback)


@dp.callback_query(F.data == "stop")
async def stop_test(callback: types.CallbackQuery):
    await show_result(callback)


async def show_result(callback: types.CallbackQuery):
    s = sessions.pop(callback.from_user.id, None)
    if s:
        answered = min(s["i"] + (1 if s["answered"] else 0), len(s["qs"]))
        text = f"Sınaq bitdi.\nNəticə: {s['score']}/{len(s['qs'])} düzgün cavab."
    else:
        text = "Sınaq bitdi."
    b = InlineKeyboardBuilder()
    b.button(text="🏠 Əsas menyu", callback_data="back_to_start")
    await callback.message.edit_text(text, reply_markup=b.as_markup())
    await callback.answer()


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
