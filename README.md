import tkinter as tk
from tkinter import filedialog, font, ttk, simpledialog, messagebox
import os
import platform
import re
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from reportlab.lib.units import mm
import numpy as np
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont


# 高DPIスケーリングを有効化
try:
    from ctypes import windll
    windll.shcore.SetProcessDpiAwareness(1)
except Exception:
    pass

class VerticalNotepad:
    def __init__(self, root):
        self.root = root
        self.root.title("縦書きメモ帳")
        self.root.geometry("600x800")

        self.current_font = font.Font(family="HiraKakuProN-W3", size=16)
        #自動改行
        self.indent_on_newline = tk.BooleanVar(value=False) 

        #テーマ変更用
        self.theme = tk.StringVar(value="Light")
        self.text_color = "black"
        self.caret_color="black"

        
        # ボタンのスタイル設定
        self.style = ttk.Style()
        self.style.configure("RoundedButton.TButton", borderwidth=0, relief="flat", padding=6, background="#e0e0e0", foreground="black")
        self.style.map("RoundedButton.TButton", background=[("active", "#c0c0c0")])
        #スクロールバーのスタイル設定
        self.style.theme_use("clam")
        self.style.configure("Vertical.TScrollbar", gripcount=0, troughcolor="#f0f0f0", background="#e0e0e0")
        self.style.map("Vertical.TScrollbar", background=[("active", "#c0c0c0")])

        self.canvas = tk.Canvas(self.root, bg="white")
        self.canvas.pack(fill="both", expand=True)

        #横方向スクロール
        self.scrollbar_x = ttk.Scrollbar(self.root, orient="horizontal", command=self.canvas.xview)
        self.scrollbar_x.pack(side=tk.BOTTOM, fill=tk.X)
        self.canvas.configure(xscrollcommand=self.scrollbar_x.set)

        #縦方向スクロール?
        # self.scrollbar = ttk.Scrollbar(self.root, orient="vertical", command=self.canvas.yview)
        # self.scrollbar.pack(side="right", fill="y")
        # self.canvas.configure(yscrollcommand=self.scrollbar.set)


        self.canvas.bind("<Configure>", self.redraw)
        self.canvas.bind("<Key>", self.on_key_press)
        self.canvas.bind("<Button-1>", self.on_mouse_click)
        self.canvas.bind("<B1-Motion>", self.on_mouse_drag)  # ドラッグイベントを追加
        self.canvas.bind("<ButtonRelease-1>", self.on_mouse_release)  # リリースイベントを追加
        self.canvas.bind("<MouseWheel>", self.on_mousewheel) 
        self.canvas.focus_set()

        self.text = ""
        self.caret_pos = 0

        
        self.create_menu()
        self.create_status_bar()
        self.apply_theme()
    
        self.search_results = []
        self.search_index = 0
        self.highlighted_ranges = []

        self.selected_text_start = None
        self.selected_text_end = None
        self.drag_start_pos = None  # ドラッグ開始位置を保持

        self.auto_indent = []

        self.search_term = ""
        self.replace_term = ""
        self.search_window_open = False
        self.search_index = 0
        self.key_pressed = False

    def create_menu(self):
        menubar = tk.Menu(self.root)
        self.root.config(menu=menubar)

        file_menu = tk.Menu(menubar, tearoff=0)
        menubar.add_cascade(label="ファイル", menu=file_menu)
        file_menu.add_command(label="新規 (Ctrl+N)", command=self.new_file, accelerator="Ctrl+N")
        file_menu.add_command(label="保存 (Ctrl+S)", command=self.save_file, accelerator="Ctrl+S")
        file_menu.add_command(label="PDF出力", command=self.export_to_pdf)
        file_menu.add_command(label="終了 (Ctrl+Q)", command=self.root.quit, accelerator="Ctrl+Q")

        edit_menu = tk.Menu(menubar, tearoff=0)
        menubar.add_cascade(label="編集", menu=edit_menu)
        edit_menu.add_command(label="コピー (Ctrl+C)", command=self.copy_text, accelerator="Ctrl+C")
        edit_menu.add_command(label="貼り付け (Ctrl+V)", command=self.paste_text, accelerator="Ctrl+V")
        edit_menu.add_command(label="切り取り (Ctrl+X)", command=self.cut_text, accelerator="Ctrl+X")
        edit_menu.add_command(label="検索・置換 (Ctrl+F)", command=self.search_text, accelerator="Ctrl+F")

        format_menu = tk.Menu(menubar, tearoff=0)
        menubar.add_cascade(label="書式", menu=format_menu)
        format_menu.add_command(label="フォント変更", command=self.change_font)
        format_menu.add_checkbutton(label="自動字下げ", variable=self.indent_on_newline)
        format_menu.add_command(label="テーマ変更", command=self.change_theme)

        self.root.bind("<Control-n>", lambda e: self.new_file())
        self.root.bind("<Control-s>", lambda e: self.save_file())
        self.root.bind("<Control-q>", lambda e: self.root.quit())
        self.root.bind("<Control-f>", lambda e: self.search_text())
        self.root.bind("<Control-h>", lambda e: self.replace_text())
        self.root.bind("<Control-c>", lambda e: self.copy_text())
        self.root.bind("<Control-v>", lambda e: self.paste_text())

    def create_status_bar(self):
        self.status_bar = tk.Label(self.root, text="", bd=1, relief=tk.SUNKEN, anchor=tk.W, bg="#e0e0e0", padx=5)
        self.status_bar.pack(side=tk.BOTTOM, fill=tk.X)
    
    # def update_status_bar(self):
    #     char_count = len(self.text)
    #     line_count = self.text.count("\n") + 1
    #     x, y = self.get_caret_coords(self.caret_pos)
    #     self.status_bar.config(text=f"文字数: {char_count}, 行数: {line_count}")

    def count_characters(self):
        char_count = len(self.text)
        line_count = self.text.count("\n") + 1
        self.status_bar.config(text=f"文字数: {char_count}, 行数: {line_count}")

    def new_file(self):
        self.text = ""
        self.caret_pos = 0
        self.redraw()
        self.highlighted_ranges = []
        self.selected_text_start = None
        self.selected_text_end = None

    def save_file(self):
        file_path = filedialog.asksaveasfilename(defaultextension=".txt",
                                               filetypes=[("Text files", "*.txt"), ("All files", "*.*")])
        if file_path:
            with open(file_path, "w", encoding="utf-8") as f:
                f.write(self.text)

    def change_font(self):
        def apply_new_font(new_font):
            self.current_font = new_font
            self.redraw()

        FontDialog(self.root, self.current_font, apply_new_font)
    
    def change_theme(self):
        themes = ["Light", "Dark","優しい", "原稿用紙風","原稿用紙風-優しい"]
        def apply_new_theme(new_theme):
            self.theme.set(new_theme)
            self.apply_theme()
            self.redraw()

        ThemeDialog(self.root, self.theme.get(), apply_new_theme, themes)

    def apply_theme(self):
        if self.theme.get() == "Dark":
            self.root.config(bg="gray12")
            self.canvas.config(bg="gray12")
            self.status_bar.config(bg="gray20", fg="white")
            self.text_color = "white"
            self.caret_color="white"
            self.canvas.update()
        elif self.theme.get() == "優しい":
            self.root.config(bg="ivory")
            self.canvas.config(bg="ivory")
            self.status_bar.config(bg="ivory", fg="ivory")
            self.text_color = "gray"
            self.caret_color="gray"
            self.canvas.update()
        elif self.theme.get() == "原稿用紙風":
            self.root.config(bg="#f8f8f8")  # 薄いグレーの背景
            self.canvas.config(bg="#f8f8f8")
            self.status_bar.config(bg="#e0e0e0", fg="black")
            self.text_color = "black"
            self.caret_color = "black"
            self.canvas.update()
        elif self.theme.get() == "原稿用紙風-優しい":
            self.root.config(bg="ivory")  # 薄いグレーの背景
            self.canvas.config(bg="ivory")
            self.status_bar.config(bg="SystemButtonFace", fg="black")
            self.text_color = "black"
            self.caret_color = "black"
            self.canvas.update()
        else:
            self.root.config(bg="white")
            self.canvas.config(bg="white")
            self.status_bar.config(bg="SystemButtonFace", fg="black")
            self.canvas.itemconfig("text", fill="black")
            self.text_color = "black"
            self.caret_color="black"
            self.canvas.update()
        

    def redraw(self, event=None):
        self.canvas.delete("all")
        width = self.canvas.winfo_width()
        height = self.canvas.winfo_height()

        line_height = self.current_font.metrics("linespace")
        char_width = self.current_font.measure("あ")
        x = width-char_width
        y = line_height

        rotate_chars = "「『（【《」』）】》―ー"
        kinsoku_chars = "、。」"

        last_char = ""
        char_index = 0
        count_return = 0

        max_x = width
        #原稿用紙風テーマに設定時のみ
        if self.theme.get() == "原稿用紙風":
            self.draw_genkou_yoshi_background(width, height, char_width, line_height , max_x, "#a52a2a")

        if self.theme.get() == "原稿用紙風-優しい":
            self.draw_genkou_yoshi_background(width, height, char_width, line_height , max_x, "#a52a2a")

        for char in self.text:
            #原稿用紙風テーマ無限に線が引けるけど重くなる
            # if self.theme.get() == "原稿用紙風":
            #     self.draw_genkou_yoshi_background(width, height, char_width, line_height , max_x)
            if char == "\n":
                #if self.indent_on_newline.get() and last_char == ".":
                #    self.text = self.text[:char_index + 1] + " " + self.text[char_index + 1:]
                #y = line_height
                #x -= char_width * 1.5
                #char_index += 1
                #continue
                if self.auto_indent[count_return]: #機能がONかOFFか
                    y = line_height*2
                    x -= char_width * 1.5
                    char_index += 1    
                else:
                    y = line_height
                    x -= char_width * 1.5
                    char_index += 1
                count_return += 1
                continue
            last_char = char

            if y == line_height:
                if char in kinsoku_chars:
                    y = height-line_height
                    x += char_width * 1.5
                
            offset_x = 0
            offset_y = 0
            angle = 0

            if char in rotate_chars:
                angle = -90
                offset_x = char_width // 4
                offset_y = line_height // 4
            elif char in "、。":
                offset_x = char_width // 2
                offset_y = -line_height // 4
            elif char in "「『（［｛":
                offset_y = -line_height // 4
            elif char in "」』）］｝":
                offset_y = line_height // 4

            # 選択範囲のハイライト表示
            if self.selected_text_start is not None and self.selected_text_end is not None:
                if self.selected_text_start <= char_index < self.selected_text_end:
                    self.canvas.create_rectangle(x - char_width // 2, y, x + char_width // 2, y + line_height, fill="lightblue", outline="")

            for start, end in self.highlighted_ranges:
                if start <= char_index < end:
                    self.canvas.create_rectangle(x - char_width // 2, y, x + char_width // 2, y + line_height, fill="yellow", outline="")

            

            self.canvas.create_text(
                x + offset_x, y + offset_y,
                text=char,
                font=self.current_font,
                anchor="center",
                angle=angle,
                fill=self.text_color, 
            )

            if char_index == self.caret_pos:
                caret_x = x
                caret_y = y
                self.canvas.create_line(caret_x - char_width // 2, caret_y, caret_x + char_width // 2, caret_y, fill=self.caret_color)

            y += line_height
            #if y > height - line_height:
            #    self.text = self.text[:char_index] + "\n" + self.text[char_index:]
            #    self.caret_pos += 1
            #    return self.redraw()
            if y > height - line_height:
                y = line_height
                x -= char_width * 1.5
            max_x = min(max_x, x)
            char_index += 1

        if self.caret_pos == len(self.text):
            caret_x = x
            caret_y = y
            self.canvas.create_line(caret_x - char_width // 2, caret_y, caret_x + char_width // 2, caret_y, fill=self.caret_color)

        self.canvas.configure(scrollregion=(max_x - width*2, 0, width, self.canvas.bbox("all")[3]))
        self.count_characters()
    

    def draw_genkou_yoshi_background(self, width, height, char_width, line_height , max_x,color):
         # 罫線の間隔を計算
        char_width = self.current_font.measure("あ")
        vertical_line_spacing = self.current_font.measure("あ") * 1.5
        line_height = self.current_font.metrics("linespace")
        # 罫線の色
        line_color = color
        ## 縦線を描画
        start_x = width -char_width+ vertical_line_spacing/2
        if max_x >= 0 :
            left_limit = max_x - width*4
        else:
            left_limit = max_x * 10 - width*4
        while start_x > left_limit:
            # 1本目の縦線を描画
            self.canvas.create_line(start_x - 1, 0, start_x - 1, height, fill=line_color)
            # 2本目の縦線を描画
            self.canvas.create_line(start_x + 1, 0, start_x + 1, height, fill=line_color)
            start_x -= vertical_line_spacing
        # 横線を描画
        start_y = line_height/2
        while start_y < height :
            self.canvas.create_line(left_limit, start_y, width, start_y, fill=line_color, dash=(2, 2))
            start_y += line_height


    def on_key_press(self, event):
        if event.keysym in ("Left", "Right", "Up", "Down"):
            self.move_caret(event.keysym)
        elif event.keysym == "Return":
            newline_index = self.text[:self.caret_pos].count("\n")
            if self.indent_on_newline.get():
                self.auto_indent.insert(newline_index, True)
            else:
                self.auto_indent.insert(newline_index, False)
            self.text = self.text[:self.caret_pos] + "\n" + self.text[self.caret_pos:]
            self.caret_pos += 1
        elif event.keysym == "space":
            self.text = self.text[:self.caret_pos] + "\u3000" + self.text[self.caret_pos:]
            self.caret_pos += 1
        elif event.char and (event.char.isprintable() or event.char == "\u3000"):
            self.text = self.text[:self.caret_pos] + event.char + self.text[self.caret_pos:]
            self.caret_pos += 1
            self.key_pressed = True
        elif event.keysym == "BackSpace" and self.caret_pos > 0:
            if self.text[self.caret_pos - 1] == "\n": #改行を削除するなら
                newline_index = self.text[:self.caret_pos].count("\n")
                del self.auto_indent[newline_index - 1]
            self.text = self.text[:self.caret_pos - 1] + self.text[self.caret_pos:]
            self.caret_pos -= 1
        elif event.keysym == "Delete" and self.caret_pos < len(self.text):
            if self.text[self.caret_pos] == "\n": #改行を削除するなら
                newline_index = self.text[:self.caret_pos].count("\n")
                del self.auto_indent[newline_index - 1]
            self.text = self.text[:self.caret_pos] + self.text[self.caret_pos + 1:]
        self.redraw()
        if self.search_window_open:
            self.perform_search()

    def move_caret(self, direction):
        if direction == "Up" and self.caret_pos > 0:
            self.caret_pos -= 1
        elif direction == "Down" and self.caret_pos < len(self.text):
            self.caret_pos += 1
        elif direction == "Left":
            line_height = self.current_font.metrics("linespace")
            char_width = self.current_font.measure("あ")
            pos = self.caret_pos
            x, y = self.get_caret_coords(pos)
            if x < self.canvas.winfo_width() - char_width:
                while pos < len(self.text) and self.get_caret_coords(pos)[0] > x - char_width:
                    pos += 1
                if pos < len(self.text):
                    self.caret_pos = pos
        elif direction == "Right":
            line_height = self.current_font.metrics("linespace")
            char_width = self.current_font.measure("あ")
            pos = self.caret_pos
            x, y = self.get_caret_coords(pos)
            if x > char_width:
                while pos > 0 and self.get_caret_coords(pos)[0] < x + char_width:
                    pos -= 1
                if pos >= 0:
                    self.caret_pos = pos
        self.redraw()

    def get_caret_coords(self, pos):
        line_height = self.current_font.metrics("linespace")
        char_width = self.current_font.measure("あ")
        x = self.canvas.winfo_width() - char_width
        y = line_height
        char_index = 0
        for char in self.text:
            if char_index == pos:
                return x, y
            if char == "\n":
                y = line_height
                x -= char_width * 1.5
            else:
                y += line_height
                if y > self.canvas.winfo_height() - line_height:
                    y = line_height
                    x -= char_width * 1.5
            char_index += 1
        return x, y

    def on_mouse_click(self, event):
        self.key_pressed = True
        self.drag_start_pos = self.get_char_index_from_coords(event.x, event.y)
        self.caret_pos = self.drag_start_pos
        self.selected_text_start = self.caret_pos
        self.selected_text_end = self.caret_pos
        self.redraw()

    def on_mouse_drag(self, event):
        if self.drag_start_pos is not None:
            current_pos = self.get_char_index_from_coords(event.x, event.y)
            self.caret_pos = current_pos
            self.selected_text_start = min(self.drag_start_pos, current_pos)
            self.selected_text_end = max(self.drag_start_pos, current_pos)
            self.redraw()

    def on_mouse_release(self, event):
        self.drag_start_pos = None

    def on_mousewheel(self, event):
        if event.delta:
            self.canvas.xview_scroll(int(-1 * (event.delta / 120)), "units")

    def get_char_index_from_coords(self, x, y):
        line_height = self.current_font.metrics("linespace")
        char_width = self.current_font.measure("あ")
        start_x = self.canvas.winfo_width() - char_width
        start_y = line_height
        char_index = 0

        for char in self.text:
            if x >= start_x - char_width // 2 and x <= start_x + char_width // 2 and y >= start_y and y <= start_y + line_height:
                return char_index
            if char == "\n":
                start_y = line_height
                start_x -= char_width * 1.5
            else:
                start_y += line_height
                if start_y > self.canvas.winfo_height() - line_height:
                    start_y = line_height
                    start_x -= char_width * 1.5
            char_index += 1

        return len(self.text)

    def search_text(self):
        #search_term = simpledialog.askstring("検索", "検索文字列を入力してください (正規表現可):")
        def on_search_change(name, index, mode):
            self.search_term = search_var.get()
            self.perform_search()
            update_search_status()

        def on_replace_change(name, index, mode):
            self.replace_term = replace_var.get()

        def on_search_window_destroy(event):
            self.highlighted_ranges = []
            self.redraw()
            self.search_window_open = False
        def next_search_result():
            if self.search_results:
                self.search_index = (self.search_index + 1) % len(self.search_results)
                self.caret_pos = self.search_results[self.search_index]
                self.redraw()
                update_search_status()

        def prev_search_result():
            if self.search_results:
                self.search_index = (self.search_index - 1) % len(self.search_results)
                self.caret_pos = self.search_results[self.search_index]
                self.redraw()
                update_search_status()
        
        def replace_text():
            if self.search_term and self.replace_term:
                try:
                    self.text, replace_count = re.subn(self.search_term, self.replace_term, self.text)
                    self.redraw()
                    self.perform_search()
                    update_search_status()
                except re.error as e:
                    messagebox.showerror("正規表現エラー", f"無効な正規表現です: {e}")

        def update_search_status():
            if self.search_results:
                status_label.config(text=f"{self.search_index + 1}/{len(self.search_results)}件")
            else:
                status_label.config(text="0件")

        search_window = tk.Toplevel(self.root)
        search_window.title("検索") # ウィンドウの名前を変更
        tk.Label(search_window, text="検索文字列を入力してください (正規表現可):").pack(padx=10, pady=5) # ラベルを追加
        search_var = tk.StringVar()
        search_entry = tk.Entry(search_window, textvariable=search_var)
        search_entry.pack(padx=10, pady=10)
        search_entry.focus_set()
        search_var.trace_add("write", on_search_change)

        tk.Label(search_window, text="置換文字列:").pack(padx=10, pady=5)
        replace_var = tk.StringVar()
        replace_entry = tk.Entry(search_window, textvariable=replace_var)
        replace_entry.pack(padx=10, pady=5)
        replace_var.trace_add("write", on_replace_change)

        button_frame = tk.Frame(search_window)
        button_frame.pack(padx=10, pady=5)

        prev_button = tk.Button(button_frame, text="<", command=prev_search_result)
        prev_button.pack(side=tk.LEFT)

        status_label = tk.Label(button_frame, text="0件")
        status_label.pack(side=tk.LEFT)

        next_button = tk.Button(button_frame, text=">", command=next_search_result)
        next_button.pack(side=tk.LEFT)

        replace_button = tk.Button(button_frame, text="置換", command=replace_text)
        replace_button.pack(side=tk.LEFT)

        self.search_window_open = True
        search_window.bind("<Destroy>", on_search_window_destroy)
    
    def perform_search(self):
        if self.search_term:
            try:
                self.search_results = [m.start() for m in re.finditer(self.search_term, self.text)]
                self.search_index = 0
                self.highlighted_ranges = []
                if self.search_results:
                    if not self.key_pressed:
                        self.caret_pos = self.search_results[0]
                    else:
                        self.key_pressed = False
                    for start_pos in self.search_results:
                        self.highlighted_ranges.append((start_pos, start_pos + len(re.search(self.search_term, self.text[start_pos:]).group())))
                    self.redraw()
                else:
                    self.highlighted_ranges = []
                    self.redraw()
            except re.error as e:
                self.highlighted_ranges = [] # 検索文字列が空の場合はハイライト表示をクリア
                self.redraw()
        else:
            self.highlighted_ranges = []
            self.redraw()

    def export_to_pdf(self):
                # フォントを登録
        font_family = self.current_font.actual()["family"]
        # font_path = f"C:/Windows/Fonts/{font_family}.ttf"  # フォントファイルのパスを指定
        # if not os.path.exists(font_path):
        #     messagebox.showerror("エラー", f"フォントファイルが見つかりません: {font_path}")
        #     return
        # try:
        #     pdfmetrics.registerFont(TTFont(font_family, font_path))
        # except Exception as e:
        #     messagebox.showerror("エラー", f"フォント登録中にエラーが発生しました: {e}")
        #     return
        file_path = filedialog.asksaveasfilename(defaultextension=".pdf",
                                               filetypes=[("PDF files", "*.pdf"), ("All files", "*.*")])
        if file_path:
            c = canvas.Canvas(file_path, pagesize=A4)
            width, height = A4
            line_height = self.current_font.metrics("linespace")
            char_width = self.current_font.measure("あ")
            x = width-char_width 
            y = height 
            char_index = 0
            count_return = 0
            rotate_chars = "「『（【《」』）】》―ー"
            page_char_count = 0
            def set_pdf_font(canvas_obj, font_family, font_size):
                    font_path = self.current_font.actual().get("file")
                    if font_path and os.path.exists(font_path):
                        try:
                            pdfmetrics.registerFont(TTFont(font_family, font_path))
                            canvas_obj.setFont(font_family, font_size)
                            return
                        except Exception as e:
                            print(f"Error registering font: {e}")
                    canvas_obj.setFont("Helvetica", font_size)
                    print(f"Using default font (Helvetica) for {font_family}")
            set_pdf_font(c, self.current_font.actual()["family"], self.current_font.actual()["size"])



            for char in self.text:
                if char == "\n":
                    y = height-line_height 
                    x -= char_width 
                    char_index += 1   
                    continue

                offset_x = 0
                offset_y = 0
                angle = 0
                if char in rotate_chars:
                    angle=-90
                    offset_x = char_width//4
                    offset_y = line_height//4 
                elif char in "、。":
                    offset_x = char_width // 2
                    offset_y = -line_height // 4
                elif char in "「『（［｛":
                    offset_y = -line_height // 4
                elif char in "」』）］｝":
                    offset_y = line_height // 4

                offset_x *= mm
                offset_y *= mm

                y -= line_height 
                if y < 0 :
                    y = height-line_height 
                    x -= char_width * 1.5 
                c.saveState()
                c.translate(x + offset_x, y + offset_y)
                c.rotate(angle)
                c.drawString(0, 0, char)
                c.restoreState()

                
                char_index+=1
                page_char_count += 1


            c.save()
            messagebox.showinfo("PDF出力", "PDFファイルを出力しました。")

    def copy_text(self):
        if self.selected_text_start is not None and self.selected_text_end is not None:
            selected_text = self.text[self.selected_text_start:self.selected_text_end]
            self.root.clipboard_clear()
            self.root.clipboard_append(selected_text)
    
    def cut_text(self):
        if self.selected_text_start is not None and self.selected_text_end is not None:
            selected_text = self.text[self.selected_text_start:self.selected_text_end]
            self.root.clipboard_clear()
            self.root.clipboard_append(selected_text)
            self.text = self.text[:self.selected_text_start] + self.text[self.selected_text_end:]
            self.caret_pos = self.selected_text_start
            self.selected_text_start = None
            self.selected_text_end = None
            self.redraw()

    def paste_text(self):
        pasted_text = self.root.clipboard_get()
        self.text = self.text[:self.caret_pos] + pasted_text + self.text[self.caret_pos:]
        self.caret_pos += len(pasted_text)
        self.redraw()

class FontDialog(tk.Toplevel):
    def __init__(self, parent, current_font, apply_callback):
        super().__init__(parent)
        self.title("フォント選択")
        self.transient(parent)
        self.result = None
        self.apply_callback = apply_callback

        tk.Label(self, text="フォント:").grid(row=0, column=0, padx=5, pady=5)
        self.font_var = tk.StringVar(value=current_font.actual()["family"])

        # フォントリストから"@"が付いているフォントを除外
        font_names = [f for f in font.families() if not f.startswith("@")]
        self.font_combo = ttk.Combobox(self, textvariable=self.font_var, values=font_names)
        self.font_combo.grid(row=0, column=1, padx=5, pady=5)

        tk.Label(self, text="サイズ:").grid(row=1, column=0, padx=5, pady=5)
        self.size_var = tk.IntVar(value=current_font.actual()["size"])
        self.size_combo = ttk.Combobox(self, textvariable=self.size_var, values=[12, 14, 16, 18, 20, 24, 28, 32])
        self.size_combo.grid(row=1, column=1, padx=5, pady=5)

        tk.Button(self, text="OK", command=self.on_ok).grid(row=2, column=0, pady=10)
        tk.Button(self, text="キャンセル", command=self.destroy).grid(row=2, column=1, pady=10)

        self.grab_set()
        self.geometry(f"+{parent.winfo_x() + 50}+{parent.winfo_y() + 50}")

    def on_ok(self):
        selected_font = self.font_var.get()
        selected_size = self.size_var.get()

        if selected_font and selected_size:
            self.result = font.Font(family=selected_font, size=selected_size)
            self.apply_callback(self.result)

        self.destroy()

class ThemeDialog(tk.Toplevel):
    def __init__(self, parent, current_theme, apply_callback, themes):
        super().__init__(parent)
        self.title("テーマ選択")
        self.transient(parent)
        self.apply_callback = apply_callback
        self.theme_var = tk.StringVar(value=current_theme)

        tk.Label(self, text="テーマ:").grid(row=0, column=0, padx=5, pady=5)
        self.theme_combo = ttk.Combobox(self, textvariable=self.theme_var, values=themes)
        self.theme_combo.grid(row=0, column=1, padx=5, pady=5)

        tk.Button(self, text="OK", command=self.on_ok).grid(row=1, column=0, pady=10)
        tk.Button(self, text="キャンセル", command=self.destroy).grid(row=1, column=1, pady=10)

        self.grab_set()
        self.geometry(f"+{parent.winfo_x() + 50}+{parent.winfo_y() + 50}")

    def on_ok(self):
        selected_theme = self.theme_var.get()
        self.apply_callback(selected_theme)
        self.destroy()

if __name__ == "__main__":
    root = tk.Tk()
    app = VerticalNotepad(root)
    root.mainloop()
