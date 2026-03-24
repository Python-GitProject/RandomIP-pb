# Смена IP для Windows

## Инструкция
1. Перейти в по пути RandomIP\tor-expert-bundle-windows-x86_64-15.0.7\tor
2. Открываем в текущем пути PowerShell
3. Генерируем хэш пароля
```powershell
.\tor.exe --hash-password "password"
```
```powershell
// пример вывода
16:7A19DCD3E4DCA171234141123424HGKW661HJ2JK379HUNB1
```
4. Копируем хэш пароля
5. В PowerShell вводим следующую команду
```powershell
new-item "torrc"
```
6. Создатся файл torrc без расширения, открываем его как .txt
7. В нем копируем следующий текст
```
SocksPort 9050
ControlPort 9061
CookieAuthentication 0
HashedControlPassword 16:7A19DCD3E4DCA171234141123424HGKW661HJ2JK379HUNB1 (здесь ваш хэш)
```
8. Далее переходим по пути RandomIP\soft и устанавливаем python в зависимости от вашей разрядности
9. После утановки в Powershell обновляем пакеты pip и устанавливаем pysocks
```powershell
python -m pip install --upgrade pip
...
pip install stem requests pysocks
```
10. На всякий случай импортируем tkinter
```powershell
python -c "import tkinter; print('ok')"
```
11. Далее переходим по пути RandomIP\soft и устанавливаем Clash.for.Windows путь для установки выбираем сами
12. Переходим в RandomIP\tor-expert-bundle-windows-x86_64-15.0.7\tor и в Powershell вбиваем команду
```powershell
new-item "Global"
```
Вставляем следующий текст в файл Global и сохраняем
```
port: 7890
socks-port: 7891
allow-lan: false
mode: Global
log-level: info
external-controller: 127.0.0.1:9090

proxies:
  - name: "Tor"
    type: socks5
    server: 127.0.0.1
    port: 9050

proxy-groups:
  - name: "PROXY"
    type: select
    proxies: 
      - "Tor"

rules:
 - MATCH,PROXY
```
13. Запускаем  Clash.for.Windows, переходим на вкладку profiles(1), нажимаем на import(2) и импортируем файл Global (3)
![Sample image](img\1.png)
14. Кликаем по импортированному конфигу Global, чтобы он загорелся зеленым
15. Переходим во вкладку Proxies и выбираем Tor
![Sample image](img\2.png)
16. Далее переходим в RandomIP\tor-expert-bundle-windows-x86_64-15.0.7\tor и создаем файл .py с названием rotator
17. В файл rotator.py вставляем следующий код и подставляем свои значения, где указаны комментарии
```python
import subprocess
import time
import threading
import requests
import tkinter as tk
from tkinter import messagebox
from stem import Signal
from stem.control import Controller
import traceback
import os
 
# Настройки
TOR_PATH = r"\tor.exe"                             # путь к tor.exe
TOR_RC = r"\torrc"                                 # путь к torrc
CLASH_PATH = r"Clash for Windows.exe"              # путь к приложению
CONTROL_PASSWORD = ""                              # пароль ControlPort (torrc)
CONTROL_PORT = 9061                                # ControlPort из torrc
CLASH_PORT = 7890                                  # локальный порт Clash (обычно 7890)
DEFAULT_INTERVAL = 60                              # секунд
CREATE_NO_WINDOW = 0x08000000
 
tor_process = None
clash_process = None
enabled = True
interval_sec = DEFAULT_INTERVAL
 
# Управление системным прокси 
def enable_system_proxy(port):
    try:
        subprocess.run([
            "reg", "add",
            r"HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings",
            "/v", "ProxyEnable", "/t", "REG_DWORD", "/d", "1", "/f"
        ], check=True)
        subprocess.run([
            "reg", "add",
            r"HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings",
            "/v", "ProxyServer", "/t", "REG_SZ", "/d", f"127.0.0.1:{port}", "/f"
        ], check=True)
    except Exception as e:
        print("Ошибка включения системного прокси:", e)
        traceback.print_exc()
 
def disable_system_proxy():
    try:
        subprocess.run([
            "reg", "add",
            r"HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings",
            "/v", "ProxyEnable", "/t", "REG_DWORD", "/d", "0", "/f"
        ], check=True)
    except Exception as e:
        print("Ошибка выключения системного прокси:", e)
        traceback.print_exc()
 
# Управление процессами
def start_tor():
    global tor_process
    if not tor_process or tor_process.poll() is not None:
        tor_process = subprocess.Popen(
            [TOR_PATH, "-f", TOR_RC],
            creationflags=subprocess.CREATE_NO_WINDOW
        )
 
def stop_tor():
    global tor_process
    if tor_process and tor_process.poll() is None:
        tor_process.terminate()
        tor_process = None
 
def start_clash():
    global clash_process
    if not clash_process or clash_process.poll() is not None:
        clash_process = subprocess.Popen(
            [CLASH_PATH],
            creationflags=CREATE_NO_WINDOW
        )
        enable_system_proxy(CLASH_PORT)
 
def stop_clash():
    global clash_process
    if clash_process and clash_process.poll() is None:
        clash_process.terminate()
        clash_process = None
    disable_system_proxy()
 
# --- Получение IP через Clash ---
def get_ip_via_clash():
    proxies = {"http": f"http://127.0.0.1:{CLASH_PORT}",
               "https": f"http://127.0.0.1:{CLASH_PORT}"}
    try:
        r = requests.get("https://api.ipify.org", proxies=proxies, timeout=10)
        return r.text.strip()
    except Exception:
        return "—"
 
# Смена IP
def change_ip():
    try:
        with Controller.from_port(port=CONTROL_PORT) as c:
            c.authenticate(password=CONTROL_PASSWORD)
            c.signal(Signal.NEWNYM)
        new_ip = get_ip_via_clash()
        root.after(0, lambda: ip_var.set("Текущий IP: " + new_ip))
    except Exception as e:
        root.after(0, lambda: ip_var.set("Ошибка смены IP"))
        print("change_ip error:", e)
        traceback.print_exc()
 
# --- GUI ---
root = tk.Tk()
root.title("Tor + Clash Rotator")
root.geometry("480x220")
 
ip_var = tk.StringVar(value="Текущий IP: —")
interval_var = tk.StringVar(value=str(DEFAULT_INTERVAL))
state_var = tk.StringVar(value="Ротация: Включена")
 
ip_label = tk.Label(root, textvariable=ip_var, font=("Segoe UI", 10))
ip_label.pack(pady=6)
 
state_label = tk.Label(root, textvariable=state_var, font=("Segoe UI", 11), fg="green")
state_label.pack()
 
frame = tk.Frame(root)
frame.pack(pady=6)
 
def toggle():
    global enabled
    enabled = not enabled
    if enabled:
        start_tor()
        start_clash()
        state_var.set("Ротация: Включена")
        state_label.config(fg="green")
    else:
        stop_tor()
        stop_clash()
        state_var.set("Ротация: Выключена")
        state_label.config(fg="red")
 
def apply_interval():
    global interval_sec
    try:
        v = int(interval_var.get())
        if 3 <= v <= 600:
            interval_sec = v
            messagebox.showinfo("OK", f"Интервал установлен: {interval_sec} сек.")
        else:
            messagebox.showwarning("Ошибка", "Интервал от 3 до 600 секунд.")
    except:
        messagebox.showwarning("Ошибка", "Введите целое число секунд.")
 
tk.Button(frame, text="Вкл/Выкл ротацию", width=16, command=toggle).grid(row=0, column=0, padx=6)
tk.Label(frame, text="Интервал (сек):").grid(row=0, column=1, padx=6)
tk.Entry(frame, width=6, textvariable=interval_var).grid(row=0, column=2)
tk.Button(frame, text="Установить", command=apply_interval).grid(row=0, column=3, padx=6)
 
tk.Button(root, text="Сменить IP сейчас", command=change_ip).pack(pady=6)
 
# Фоновая ротация
def rotator_thread():
    global enabled
    while True:
        if enabled:
            change_ip()
        time.sleep(interval_sec)
 
t = threading.Thread(target=rotator_thread, daemon=True)
t.start()
 
def on_close():
    stop_tor()
    stop_clash()
    root.destroy()
 
root.protocol("WM_DELETE_WINDOW", on_close)
 
# Автозапуск
start_tor()
start_clash()
change_ip()
 
root.mainloop()
```
18. Запускаем rotator.py 

