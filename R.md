import sqlite3
import tkinter as tk
from tkinter import ttk, messagebox
from collections import OrderedDict

# BẢNG BĂM CACHE 
class QueryCache:
    def __init__(self, capacity=3):
        self.cache = OrderedDict()
        self.capacity = capacity
        self.conn = sqlite3.connect("students.db")

    def execute_query(self, query, params=()):
        key = (query, params)
        if key in self.cache:
            self.cache.move_to_end(key)
            print("Cache HIT")
            return self.cache[key]
        else:
            print("Cache MISS")
            cursor = self.conn.execute(query, params)
            result = cursor.fetchall()
            self.cache[key] = result
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)
            return result

# TẠO DATABASE 
def setup_database():
    conn = sqlite3.connect("students.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS students (
            id INTEGER PRIMARY KEY,
            name TEXT,
            major TEXT
        )
    """)
    cursor.execute("DELETE FROM students")  # Xóa dữ liệu cũ 

    # Thêm dữ liệu mẫu
    data = [
        (1, "Nguyen Van A", "CNTT"),
        (2, "Tran Thi B", "Kinh tế"),
        (3, "Le Van C", "CNTT"),
        (4, "Pham Thi D", "Y học"),
        (5, "Hoang Van E", "Kế toán")
    ]
    cursor.executemany("INSERT INTO students VALUES (?, ?, ?)", data)
    conn.commit()
    conn.close()

# GIAO DIỆN 
def launch_gui():
    cache = QueryCache(capacity=3)

    def search_student():
        query_type = combo_type.get()
        keyword = entry_input.get()

        try:
            if query_type == "Theo ID":
                student_id = int(keyword)
                query = "SELECT * FROM students WHERE id = ?"
                params = (student_id,)
            elif query_type == "Theo Tên":
                query = "SELECT * FROM students WHERE name LIKE ?"
                params = (f"%{keyword}%",)
            elif query_type == "Theo Ngành học":
                query = "SELECT * FROM students WHERE major LIKE ?"
                params = (f"%{keyword}%",)
            elif query_type == "Tất cả":
                query = "SELECT * FROM students"
                params = ()
            else:
                raise ValueError("Chưa chọn kiểu truy vấn.")

            result = cache.execute_query(query, params)
            if result:
                output = "\n".join([f"ID: {r[0]}, Tên: {r[1]}, Ngành: {r[2]}" for r in result])
            else:
                output = "Không tìm thấy sinh viên nào."

            result_text.configure(state='normal')
            result_text.delete("1.0", tk.END)
            result_text.insert(tk.END, output)
            result_text.configure(state='disabled')

        except ValueError:
            messagebox.showerror("Lỗi", "ID phải là số nguyên.")
        except Exception as e:
            messagebox.showerror("Lỗi", str(e))

    root = tk.Tk()
    root.title("Tra cứu Sinh viên ")
    root.geometry("550x400")
    root.configure(bg="#f0f4f8")
    root.resizable(False, False)

    style = ttk.Style()
    style.theme_use("clam")
    style.configure("TLabel", background="#f0f4f8", font=("Arial", 11))
    style.configure("TButton", font=("Arial", 11), padding=5)
    style.configure("TCombobox", font=("Arial", 11))

    frame = ttk.Frame(root, padding=20)
    frame.pack(fill=tk.BOTH, expand=True)

    ttk.Label(frame, text="Loại tra cứu:").grid(row=0, column=0, sticky=tk.W, pady=5)
    combo_type = ttk.Combobox(frame, values=["Theo ID", "Theo Tên", "Theo Ngành học", "Tất cả"], state="readonly")
    combo_type.grid(row=0, column=1, sticky=tk.EW, pady=5)
    combo_type.current(0)

    ttk.Label(frame, text="Nhập từ khóa:").grid(row=1, column=0, sticky=tk.W, pady=5)
    entry_input = ttk.Entry(frame, font=("Arial", 11))
    entry_input.grid(row=1, column=1, sticky=tk.EW, pady=5)

    ttk.Button(frame, text="Tìm kiếm", command=search_student).grid(row=2, column=0, columnspan=2, pady=10)

    ttk.Label(frame, text="Kết quả:").grid(row=3, column=0, columnspan=2, sticky=tk.W, pady=5)
    result_frame = ttk.Frame(frame)
    result_frame.grid(row=4, column=0, columnspan=2, sticky="nsew")

    result_text = tk.Text(result_frame, height=10, width=60, font=("Courier", 10), wrap=tk.WORD, state='disabled')
    result_text.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

    scrollbar = ttk.Scrollbar(result_frame, command=result_text.yview)
    scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
    result_text.config(yscrollcommand=scrollbar.set)

    frame.columnconfigure(1, weight=1)
    root.mainloop()

if __name__ == "__main__":
    setup_database()
    launch_gui()
