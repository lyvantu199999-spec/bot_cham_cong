import telebot
from telebot import types
from datetime import datetime
import sqlite3
import pandas as pd
import os

TOKEN = "8765508087:AAEjRALLHYLVeLyydN0vGTswfppfkXXLFjM"   # ← Thay token vào đây

bot = telebot.TeleBot(TOKEN)

# ====================== DATABASE ======================
conn = sqlite3.connect('chamcong.db', check_same_thread=False)
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE IF NOT EXISTS employees (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL
)
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS attendance (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    employee_id INTEGER,
    date TEXT,
    work_units REAL,
    note TEXT,
    FOREIGN KEY (employee_id) REFERENCES employees(id)
)
''')
conn.commit()

# ====================== MENU CHÍNH ======================
def main_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add("👷 Quản lý Nhân viên", "⏰ Chấm công")
    markup.add("📊 Xem công", "📤 Xuất Excel")
    return markup

@bot.message_handler(commands=['start'])
def start(message):
    bot.reply_to(message, "👋 **Bot Quản lý Chấm Công**\n\nChọn chức năng bên dưới:", 
                 reply_markup=main_menu())

# ====================== QUẢN LÝ NHÂN VIÊN ======================
@bot.message_handler(func=lambda m: m.text == "👷 Quản lý Nhân viên")
def quanly_nhanvien(message):
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(types.InlineKeyboardButton("➕ Thêm nhân viên", callback_data="add_emp"))
    markup.add(types.InlineKeyboardButton("📋 Danh sách nhân viên", callback_data="list_emp"))
    markup.add(types.InlineKeyboardButton("🗑 Xóa nhân viên", callback_data="del_emp"))
    bot.send_message(message.chat.id, "🔧 **Quản lý Nhân viên**", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data == "add_emp")
def add_emp(call):
    msg = bot.send_message(call.message.chat.id, "Nhập tên nhân viên mới:")
    bot.register_next_step_handler(msg, save_new_emp)

def save_new_emp(message):
    name = message.text.strip()
    try:
        cursor.execute("INSERT INTO employees (name) VALUES (?)", (name,))
        conn.commit()
        bot.reply_to(message, f"✅ Đã thêm: **{name}**")
    except:
        bot.reply_to(message, "❌ Tên nhân viên đã tồn tại!")

@bot.callback_query_handler(func=lambda call: call.data == "list_emp")
def list_emp(call):
    cursor.execute("SELECT id, name FROM employees ORDER BY name")
    emps = cursor.fetchall()
    if not emps:
        bot.send_message(call.message.chat.id, "Chưa có nhân viên nào.")
        return
    text = "📋 **Danh sách nhân viên:**\n\n"
    for emp in emps:
        text += f"• {emp[1]}\n"
    bot.send_message(call.message.chat.id, text)

@bot.callback_query_handler(func=lambda call: call.data == "del_emp")
def del_emp_list(call):
    cursor.execute("SELECT id, name FROM employees")
    emps = cursor.fetchall()
    if not emps:
        bot.send_message(call.message.chat.id, "Không có nhân viên nào để xóa.")
        return
    markup = types.InlineKeyboardMarkup(row_width=1)
    for emp in emps:
        markup.add(types.InlineKeyboardButton(emp[1], callback_data=f"delete_{emp[0]}"))
    bot.send_message(call.message.chat.id, "🗑 Chọn nhân viên muốn xóa:", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data.startswith("delete_"))
def confirm_delete(call):
    emp_id = int(call.data.split("_")[1])
    cursor.execute("SELECT name FROM employees WHERE id=?", (emp_id,))
    name = cursor.fetchone()[0]
    cursor.execute("DELETE FROM employees WHERE id=?", (emp_id,))
    cursor.execute("DELETE FROM attendance WHERE employee_id=?", (emp_id,))
    conn.commit()
    bot.send_message(call.message.chat.id, f"✅ Đã xóa nhân viên: **{name}** và toàn bộ công của họ.")

# ====================== CHẤM CÔNG ======================
@bot.message_handler(func=lambda m: m.text == "⏰ Chấm công")
def cham_cong_menu(message):
    cursor.execute("SELECT id, name FROM employees ORDER BY name")
    emps = cursor.fetchall()
    if not emps:
        bot.reply_to(message, "Chưa có nhân viên. Hãy thêm nhân viên trước.")
        return
    markup = types.InlineKeyboardMarkup(row_width=2)
    for emp in emps:
        markup.add(types.InlineKeyboardButton(emp[1], callback_data=f"cham_{emp[0]}"))
    bot.send_message(message.chat.id, "Chọn nhân viên để chấm công:", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data.startswith("cham_"))
def input_units(call):
    emp_id = int(call.data.split("_")[1])
    cursor.execute("SELECT name FROM employees WHERE id=?", (emp_id,))
    name = cursor.fetchone()[0]
    
    msg = bot.send_message(call.message.chat.id, 
f"Nhập số công cho **{name}** (ví dụ: 1 hoặc 0.5)\n\nBạn cũng có thể thêm ghi chú sau dấu cách.\nVí dụ: `1 Đi làm đầy đủ`")
    bot.register_next_step_handler(msg, lambda m: save_cham_cong(m, emp_id, name))

def save_cham_cong(message, emp_id, name):
    try:
        text = message.text.strip()
        parts = text.split(maxsplit=1)
        units = float(parts[0].replace(",", "."))
        note = parts[1] if len(parts) > 1 else ""
        today = datetime.now().strftime("%d/%m/%Y")
        
        cursor.execute("INSERT INTO attendance (employee_id, date, work_units, note) VALUES (?,?,?,?)",
                      (emp_id, today, units, note))
        conn.commit()
        bot.reply_to(message, f"✅ Đã chấm **{units} công** cho {name}")
    except:
        bot.reply_to(message, "❌ Sai định dạng! Vui lòng nhập số công.")

# ====================== XEM CÔNG & XUẤT FILE ======================
@bot.message_handler(func=lambda m: m.text == "📊 Xem công")
def xem_cong_menu(message):
    cursor.execute("SELECT id, name FROM employees")
    emps = cursor.fetchall()
    markup = types.InlineKeyboardMarkup(row_width=2)
    for emp in emps:
        markup.add(types.InlineKeyboardButton(emp[1], callback_data=f"view_{emp[0]}"))
    bot.send_message(message.chat.id, "Chọn nhân viên để xem công:", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data.startswith("view_"))
def view_employee(call):
    emp_id = int(call.data.split("_")[1])
    cursor.execute("SELECT name FROM employees WHERE id=?", (emp_id,))
    name = cursor.fetchone()[0]
    
    cursor.execute("""
        SELECT date, work_units, note 
        FROM attendance 
        WHERE employee_id=? 
        ORDER BY date DESC LIMIT 30
    """, (emp_id,))
    records = cursor.fetchall()
    
    if not records:
        bot.send_message(call.message.chat.id, f"{name} chưa có dữ liệu.")
        return
    
    text = f"📊 Công của **{name}**:\n\n"
    total = 0
    for r in records:
        text += f"{r[0]} → {r[1]} công {('('+r[2]+')' if r[2] else '')}\n"
        total += r[1]
    text += f"\n**Tổng cộng: {total} công**"
    bot.send_message(call.message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == "📤 Xuất Excel")
def xuat_excel(message):
    cursor.execute("""
        SELECT e.name, a.date, a.work_units, a.note
        FROM attendance a
        JOIN employees e ON a.employee_id = e.id
        ORDER BY e.name, a.date DESC
    """)
    data = cursor.fetchall()
    
    if not data:
        bot.reply_to(message, "Chưa có dữ liệu chấm công!")
        return
    
    df = pd.DataFrame(data, columns=["Tên Nhân Viên", "Ngày", "Số Công", "Ghi chú"])
    filename = f"ChamCong_{datetime.now().strftime('%m_%Y')}.xlsx"
    df.to_excel(filename, index=False)
    
    with open(filename, "rb") as f:
        bot.send_document(message.chat.id, f, caption=f"✅ File chấm công tháng {datetime.now().strftime('%m/%Y')}")
    
    os.remove(filename)

# ====================== CHẠY BOT ======================
print("🚀 Bot Quản lý Chấm Công đang chạy...")
bot.infinity_polling()