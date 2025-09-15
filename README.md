import tkinter as tk
from tkinter import messagebox
import hashlib
import random
from datetime import datetime

class UniversalSecurityKeypad:
    def __init__(self, root):
        self.root = root
        self.root.title("УНІВЕРСАЛЬНА СИСТЕМА КОНТРОЛЮ ДОСТУПУ")
        self.root.geometry("450x580")  # Трохи збільшив висоту для нових кнопок
        self.root.resizable(False, False)
        
        # Конфігураційні параметри
        self.raw_password = "3221"
        self.correct_code_hash = hashlib.sha256(self.raw_password.encode()).hexdigest()
        self.max_attempts = 5
        self.block_time_sec = 30
        
        # Стан системи
        self.attempts = 0
        self.is_blocked = False
        self.input_sequence = []
        self.shuffle_mode = False
        
        # Ініціалізація інтерфейсу
        self.setup_ui()

    def setup_ui(self):
        main_frame = tk.Frame(self.root, padx=10, pady=10)
        main_frame.pack(fill=tk.BOTH, expand=True)
        
        # Заголовок (зменшив трошки розмір)
        title_label = tk.Label(main_frame, text="СИСТЕМА КОНТРОЛЮ ДОСТУПУ", 
                              font=("Arial", 14, "bold"), fg="darkblue")
        title_label.pack(pady=3)
        
        # Інформаційна панель
        info_frame = tk.Frame(main_frame)
        info_frame.pack(pady=5, fill=tk.X)
        
        self.attempts_label = tk.Label(info_frame, text=f"Спроби: {self.attempts}/{self.max_attempts}", 
                                      font=("Arial", 9), fg="green")
        self.attempts_label.pack(side=tk.LEFT)
        
        self.status_label = tk.Label(info_frame, text="Статус: Очікую код...", 
                                    font=("Arial", 9), fg="gray")
        self.status_label.pack(side=tk.RIGHT)
        
        # Поле вводу коду
        entry_frame = tk.Frame(main_frame)
        entry_frame.pack(pady=8, fill=tk.X)
        
        self.entry = tk.Entry(entry_frame, width=20, font=("Arial", 14), 
                             justify='center', show="•")
        self.entry.pack(pady=4)
        self.entry.bind("<Return>", lambda e: self.check_code())
        self.entry.focus_set()
        self.entry.bind("<KeyPress>", lambda e: "break")
        
        # Клавіатура
        self.keypad_frame = tk.Frame(main_frame)
        self.keypad_frame.pack(pady=8)
        self.setup_keypad()
        
        # Кнопки управління
        control_frame = tk.Frame(main_frame)
        control_frame.pack(pady=8, fill=tk.X)
        
        self.shuffle_btn = tk.Button(control_frame, text="Перемішати клавіші", 
                                    command=self.toggle_shuffle, font=("Arial", 8))
        self.shuffle_btn.pack(side=tk.LEFT, padx=2)
        
        clear_btn = tk.Button(control_frame, text="Скинути ввід", 
                             command=self.clear_input, font=("Arial", 8))
        clear_btn.pack(side=tk.LEFT, padx=2)
        
        enter_btn = tk.Button(control_frame, text="Ввести код", 
                             command=self.check_code, font=("Arial", 8, "bold"))
        enter_btn.pack(side=tk.RIGHT, padx=2)
        
        # Кнопка скидання журналу
        clear_log_btn = tk.Button(control_frame, text="Очистити журнал", 
                                 command=self.clear_log, font=("Arial", 8))
        clear_log_btn.pack(side=tk.RIGHT, padx=2)
        
        # Лог подій
        log_frame = tk.Frame(main_frame)
        log_frame.pack(pady=8, fill=tk.BOTH, expand=True)
        
        log_label = tk.Label(log_frame, text="Журнал подій:", font=("Arial", 9))
        log_label.pack(anchor=tk.W)
        
        self.log = tk.Text(log_frame, height=6, width=40, font=("Consolas", 8))
        scrollbar = tk.Scrollbar(log_frame, orient="vertical", command=self.log.yview)
        self.log.configure(yscrollcommand=scrollbar.set)
        
        self.log.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        self.log_message("Система активована. Введіть код доступу...")
        self.log.config(state=tk.DISABLED)

    def setup_keypad(self):
        # Новий макет: 0 посередині, C зліва, E справа
        standard_layout = [
            ['7', '8', '9'],
            ['4', '5', '6'],
            ['1', '2', '3'],
            ['C', '0', 'E']  # 0 посередині, C зліва, E справа
        ]
        
        # Випадковий макет (також з 0 посередині в останньому ряду)
        digits = ['1','2','3','4','5','6','7','8','9','C','E']
        random.shuffle(digits)
        random_layout = [
            digits[0:3],
            digits[3:6],
            digits[6:9],
            ['C', '0', 'E']  # 0 завжди посередині в останньому ряду
        ]
        
        layout = random_layout if self.shuffle_mode else standard_layout
        
        # Очищаємо стару клавіатуру
        for widget in self.keypad_frame.winfo_children():
            widget.destroy()
        
        # Створюємо нову
        for i, row in enumerate(layout):
            for j, key in enumerate(row):
                btn_width = 4
                btn_height = 2
                
                if key == 'C':
                    btn = tk.Button(self.keypad_frame, text="C", width=btn_width, height=btn_height,
                                  font=("Arial", 12, "bold"), fg="red", bg="#FFCCCC",
                                  command=self.clear_input)
                elif key == 'E':
                    btn = tk.Button(self.keypad_frame, text="✓", width=btn_width, height=btn_height,
                                  font=("Arial", 12, "bold"), fg="green", bg="#CCFFCC",
                                  command=self.check_code)
                else:
                    btn = tk.Button(self.keypad_frame, text=key, width=btn_width, height=btn_height,
                                  font=("Arial", 12, "bold"), bg="#F0F0F0",
                                  command=lambda k=key: self.add_digit(k))
                
                btn.grid(row=i, column=j, padx=2, pady=2)

    def add_digit(self, digit):
        if self.is_blocked:
            self.log_message("Система заблокована! Очікуйте...", "error")
            return
            
        if self.attempts >= self.max_attempts:
            self.block_system()
            return
            
        self.input_sequence.append(digit)
        current_code = "".join(self.input_sequence)
        self.entry.delete(0, tk.END)
        self.entry.insert(0, current_code)
        
        self.log_message(f"Введено цифру: {digit}")
        
        # Автоперевірка при повному введенні
        if len(self.input_sequence) == len(self.raw_password):
            self.check_code()

    def check_code(self):
        if self.is_blocked:
            return
            
        code = "".join(self.input_sequence)
        code_hash = hashlib.sha256(code.encode()).hexdigest()
        
        if code_hash == self.correct_code_hash:
            self.log_message("УСПІХ! Код вірний. Доступ надано.", "success")
            self.status_label.config(text="Статус: Доступ надано", fg="green")
            messagebox.showinfo("Успіх", "Код вірний! Система розблокована.")
            self.reset_system()
        else:
            self.attempts += 1
            self.attempts_label.config(text=f"Спроби: {self.attempts}/{self.max_attempts}")
            self.log_message(f"Невірний код: {code}", "error")
            
            if self.attempts >= self.max_attempts:
                self.block_system()
        
        self.clear_input()

    def clear_input(self):
        self.input_sequence = []
        self.entry.delete(0, tk.END)
        self.log_message("Введення скинуто")

    def clear_log(self):
        """Очистити журнал подій"""
        self.log.config(state=tk.NORMAL)
        self.log.delete(1.0, tk.END)
        self.log_message("Журнал очищено")
        self.log.config(state=tk.DISABLED)

    def toggle_shuffle(self):
        self.shuffle_mode = not self.shuffle_mode
        if self.shuffle_mode:
            self.shuffle_btn.config(text="Стандартні клавіші", fg="red")
            self.log_message("Клавіши перемішані (режим проти клікерів)")
        else:
            self.shuffle_btn.config(text="Перемішати клавіші", fg="black")
            self.log_message("Стандартне розташування клавіш")
        
        self.setup_keypad()

    def block_system(self):
        self.is_blocked = True
        self.status_label.config(text="Статус: ЗАБЛОКОВАНО", fg="red")
        self.log_message(f"СИСТЕМА ЗАБЛОКОВАНА на {self.block_time_sec} секунд!", "error")
        messagebox.showerror("Блокування", "Забагато невдалих спроб! Система заблокована.")
        
        # Після закінчення часу блокування автоматичне відновлення
        self.root.after(self.block_time_sec * 1000, self.reset_system)

    def reset_system(self):
        self.attempts = 0
        self.is_blocked = False
        self.clear_input()
        self.attempts_label.config(text=f"Спроби: {self.attempts}/{self.max_attempts}")
        self.status_label.config(text="Статус: Очікую код...", fg="gray")
        self.log_message("Система скинута. Готово до роботи.")

    def log_message(self, message, msg_type="info"):
        """Додати повідомлення до журналу з часом"""
        timestamp = datetime.now().strftime("%H:%M:%S")
        self.log.config(state=tk.NORMAL)
        
        if msg_type == "success":
            self.log.insert(tk.END, f"[{timestamp}] УСПІХ: {message}\n")
        elif msg_type == "error":
            self.log.insert(tk.END, f"[{timestamp}] ПОМИЛКА: {message}\n")
        else:
            self.log.insert(tk.END, f"[{timestamp}] {message}\n")
        
        self.log.see(tk.END)
        self.log.config(state=tk.DISABLED)

# Запуск програми
if __name__ == "__main__":
    root = tk.Tk()
    app = UniversalSecurityKeypad(root)
    root.mainloop()￼Enterimport tkinter as tk
from tkinter import messagebox
import hashlib
import random
from datetime import datetime

class UniversalSecurityKeypad:
    def __init__(self, root):
        self.root = root
        self.root.title("УНІВЕРСАЛЬНА СИСТЕМА КОНТРОЛЮ ДОСТУПУ")
        self.root.geometry("450x580")  # Трохи збільшив висоту для нових кнопок
        self.root.resizable(False, False)
        
        # Конфігураційні параметри
        self.raw_password = "3221"
        self.correct_code_hash = hashlib.sha256(self.raw_password.encode()).hexdigest()
        self.max_attempts = 5
        self.block_time_sec = 30
        
        # Стан системи
        self.attempts = 0
        self.is_blocked = False
        self.input_sequence = []
        self.shuffle_mode = False
        
        # Ініціалізація інтерфейсу
        self.setup_ui()

    def setup_ui(self):
        main_frame = tk.Frame(self.root, padx=10, pady=10)
        main_frame.pack(fill=tk.BOTH, expand=True)
        
        # Заголовок (зменшив трошки розмір)
        title_label = tk.Label(main_frame, text="СИСТЕМА КОНТРОЛЮ ДОСТУПУ", 
                              font=("Arial", 14, "bold"), fg="darkblue")
        title_label.pack(pady=3)
        
        # Інформаційна панель
        info_frame = tk.Frame(main_frame)
        info_frame.pack(pady=5, fill=tk.X)
  
        self.attempts_label = tk.Label(info_frame, text=f"Спроби: {self.attempts}/{self.max_attempts}", 
                                      font=("Arial", 9), fg="green")
        self.attempts_label.pack(side=tk.LEFT)
        
        self.status_label = tk.Label(info_frame, text="Статус: Очікую код...", 
                                    font=("Arial", 9), fg="gray")
        self.status_label.pack(side=tk.RIGHT)
        
        # Поле вводу коду
        entry_frame = tk.Frame(main_frame)
        entry_frame.pack(pady=8, fill=tk.X)
        
        self.entry = tk.Entry(entry_frame, width=20, font=("Arial", 14), 
                             justify='center', show="•")
        self.entry.pack(pady=4)
        self.entry.bind("<Return>", lambda e: self.check_code())
        self.entry.focus_set()
        self.entry.bind("<KeyPress>", lambda e: "break")
        
        # Клавіатура
        self.keypad_frame = tk.Frame(main_frame)
        self.keypad_frame.pack(pady=8)
        self.setup_keypad()
        
        # Кнопки управління
        control_frame = tk.Frame(main_frame)
        control_frame.pack(pady=8, fill=tk.X)
        
        self.shuffle_btn = tk.Button(control_frame, text="Перемішати клавіші", 
                                    command=self.toggle_shuffle, font=("Arial", 8))
        self.shuffle_btn.pack(side=tk.LEFT, padx=2)
        
        clear_btn = tk.Button(control_frame, text="Скинути ввід", 
                             command=self.clear_input, font=("Arial", 8))
        clear_btn.pack(side=tk.LEFT, padx=2)
        
        enter_btn = tk.Button(control_frame, text="Ввести код", 
                             command=self.check_code, font=("Arial", 8, "bold"))
        enter_btn.pack(side=tk.RIGHT, padx=2)
        
        # Кнопка скидання журналу
        clear_log_btn = tk.Button(control_frame, text="Очистити журнал", 
                                 command=self.clear_log, font=("Arial", 8))
        clear_log_btn.pack(side=tk.RIGHT, padx=2)
        
        # Лог подій
        log_frame = tk.Frame(main_frame)
        log_frame.pack(pady=8, fill=tk.BOTH, expand=True)
        
        log_label = tk.Label(log_frame, text="Журнал подій:", font=("Arial", 9))
        log_label.pack(anchor=tk.W)
        
        self.log = tk.Text(log_frame, height=6, width=40, font=("Consolas", 8))
        scrollbar = tk.Scrollbar(log_frame, orient="vertical", command=self.log.yview)
        self.log.configure(yscrollcommand=scrollbar.set)
        
        self.log.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        self.log_message("Система активована. Введіть код доступу...")
        self.log.config(state=tk.DISABLED)

    def setup_keypad(self):
        # Новий макет: 0 посередині, C зліва, E справа
        standard_layout = [
            ['7', '8', '9'],
            ['4', '5', '6'],
            ['1', '2', '3'],
            ['C', '0', 'E']  # 0 посередині, C зліва, E справа
        ]
        
        # Випадковий макет (також з 0 посередині в останньому ряду)
        digits = ['1','2','3','4','5','6','7','8','9','C','E']
        random.shuffle(digits)
        random_layout = [
            digits[0:3],
            digits[3:6],
            digits[6:9],
            ['C', '0', 'E']  # 0 завжди посередині в останньому ряду
        ]
        
        layout = random_layout if self.shuffle_mode else standard_layout
        
        # Очищаємо стару клавіатуру
        for widget in self.keypad_frame.winfo_children():
            widget.destroy()
        
        # Створюємо нову
        for i, row in enumerate(layout):
            for j, key in enumerate(row):
                btn_width = 4
                btn_height = 2
                
                if key == 'C':
                    btn = tk.Button(self.keypad_frame, text="C", width=btn_width, height=btn_height,
                                  font=("Arial", 12, "bold"), fg="red", bg="#FFCCCC",
                                  command=self.clear_input)
                elif key == 'E':
                    btn = tk.Button(self.keypad_frame, text="✓", width=btn_width, height=btn_height,
                                  font=("Arial", 12, "bold"), fg="green", bg="#CCFFCC",
                                  command=self.check_code)
                else:
                    btn = tk.Button(self.keypad_frame, text=key, width=btn_width, height=btn_height,
                                  font=("Arial", 12, "bold"), bg="#F0F0F0",
