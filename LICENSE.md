import tkinter as tk  
from tkinter import ttk  
import time  

class MotoblockApp:  
    def __init__(self, root):  
        self.root = root  
        self.root.title("Симуляція мотоблока")  

        # ---------- Швидкості ----------  
        self.speed_frame = tk.Frame(root)  
        self.speed_frame.pack(pady=5, fill=tk.X)  

        tk.Label(self.speed_frame, text="робоча:", font=("Arial", 8)).grid(row=0, column=0, padx=5, sticky=tk.W)  
        self.work_speed_label = tk.Label(self.speed_frame, text="0 м", font=("Arial", 8, "bold"), fg="blue")  
        self.work_speed_label.grid(row=0, column=1, padx=5, sticky=tk.W)  

        tk.Label(self.speed_frame, text="транспортна:", font=("Arial", 8)).grid(row=0, column=2, padx=5, sticky=tk.W)  
        self.transport_speed_label = tk.Label(self.speed_frame, text="0 м", font=("Arial", 8, "bold"), fg="green")  
        self.transport_speed_label.grid(row=0, column=3, padx=5, sticky=tk.W)  

        # ---------- Панель введення параметрів ----------  
        self.frame = tk.Frame(root)  
        self.frame.pack(pady=10)  

        tk.Label(self.frame, text="Довжина (м):").grid(row=0, column=0)  
        self.length_entry = tk.Entry(self.frame)  
        self.length_entry.grid(row=0, column=1)  

        tk.Label(self.frame, text="Ширина (м):").grid(row=1, column=0)  
        self.width_entry = tk.Entry(self.frame)  
        self.width_entry.grid(row=1, column=1)  

        tk.Label(self.frame, text="Відстань між рядками (см):").grid(row=2, column=0)  
        self.row_spacing_entry = tk.Entry(self.frame)  
        self.row_spacing_entry.insert(0, "30")  
        self.row_spacing_entry.grid(row=2, column=1)  

        tk.Label(self.frame, text="Відступ від лівого краю (см):").grid(row=3, column=0)  
        self.left_margin_entry = tk.Entry(self.frame)  
        self.left_margin_entry.insert(0, "30")  
        self.left_margin_entry.grid(row=3, column=1)  

        tk.Label(self.frame, text="Відступ від правого краю (см):").grid(row=4, column=0)  
        self.right_margin_entry = tk.Entry(self.frame)  
        self.right_margin_entry.insert(0, "30")  
        self.right_margin_entry.grid(row=4, column=1)  

        tk.Label(self.frame, text="Кількість рядків:").grid(row=5, column=0)  
        self.rows_entry = tk.Entry(self.frame)  
        self.rows_entry.insert(0, "10")  
        self.rows_entry.grid(row=5, column=1)  

        tk.Label(self.frame, text="Сторона старту:").grid(row=6, column=0)  
        self.start_side_var = tk.StringVar(value="Ліва")  
        self.start_side_menu = ttk.Combobox(self.frame, textvariable=self.start_side_var, values=["Ліва", "Права"], state="readonly", width=10)  
        self.start_side_menu.grid(row=6, column=1)  

        self.follow_var = tk.BooleanVar(value=True)  
        self.follow_check = tk.Checkbutton(self.frame, text="Фіксувати мотоблок", variable=self.follow_var)  
        self.follow_check.grid(row=7, column=0, columnspan=2)  

        # ---------- Кнопки управління ----------  
        self.control_frame = tk.Frame(self.frame)  
        self.control_frame.grid(row=8, column=0, columnspan=2, pady=10)  

        self.start_btn = tk.Button(self.control_frame, text="Старт", command=self.start_simulation, width=8)  
        self.start_btn.grid(row=0, column=0, padx=5)  

        self.pause_btn = tk.Button(self.control_frame, text="Пауза", command=self.pause_simulation, width=8, state=tk.DISABLED)  
        self.pause_btn.grid(row=0, column=1, padx=5)  

        self.restart_btn = tk.Button(self.control_frame, text="Рестарт", command=self.restart_simulation, width=8)  
        self.restart_btn.grid(row=0, column=2, padx=5)  

        # ---------- Кнопка Агрегати ----------  
        self.agg_btn = tk.Button(self.control_frame, text="Агрегати ▼", command=self.toggle_aggregates_menu, width=12)  
        self.agg_btn.grid(row=0, column=3, padx=5)  

        self.aggregates_frame = tk.Frame(self.control_frame)  
        self.aggregates_visible = False  

        self.plant_btn = tk.Button(self.aggregates_frame, text="Садити", width=10, command=lambda: self.set_aggregate_mode("plant"))  
        self.plant_btn.pack(side=tk.TOP, pady=2)  

        self.dig_btn = tk.Button(self.aggregates_frame, text="Копати", width=10, command=lambda: self.set_aggregate_mode("dig"))  
        self.dig_btn.pack(side=tk.TOP, pady=2)  

        self.current_aggregate = None  

        # ---------- Канвас ----------  
        self.canvas_frame = tk.Frame(root)  
        self.canvas_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)  

        self.canvas = tk.Canvas(self.canvas_frame, bg="white", scrollregion=(0, 0, 100, 100))  
        self.canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)  

        # ---------- Службові змінні ----------  
        self.timer_label = tk.Label(root, text="Час: 00:00:00", font=("Arial", 12))  
        self.timer_label.pack(side=tk.LEFT, padx=10)  

        self.scale = 1.0  
        self.start_time = 0  
        self.timer_running = False  
        self.paused = False  
        self.total_pause_time = 0  
        self.completed_rows = 0  
        self.total_rows = 0  

    # ---------- Меню агрегатів ----------  
    def toggle_aggregates_menu(self):  
        if self.aggregates_visible:  
            self.aggregates_frame.grid_forget()  
            self.aggregates_visible = False  
        else:  
            self.aggregates_frame.grid(row=1, column=3)  
            self.aggregates_visible = True  

    def set_aggregate_mode(self, mode):  
        self.current_aggregate = mode  
        if mode == "plant":  
            color = "#556B2F"  # темно-зелений  
        elif mode == "dig":  
            color = "#1E90FF"  # синій  
        else:  
            color = "grey"  

        # Малюємо вертикальну сітку для наочності  
        self.canvas.delete("aggregate")  
        row_spacing = int(self.row_spacing_entry.get())  
        left_margin = int(self.left_margin_entry.get())  
        top_margin = 25  
        for i in range(int(self.rows_entry.get())):  
            y = top_margin + i * row_spacing  
            self.canvas.create_line(left_margin, y, int(self.width_entry.get())*100, y, fill=color, width=3, tags="aggregate")  

    # ---------- Таймер та логіка руху (як раніше) ----------  
    def start_simulation(self):  
        pass  # твоя існуюча логіка старту  

    def pause_simulation(self):  
        pass  # твоя існуюча логіка паузи  

    def restart_simulation(self):  
        pass  # твоя існуюча логіка рестарту  

if __name__ == "__main__":  
    root = tk.Tk()  
    app = MotoblockApp(root)  
    root.mainloop()
