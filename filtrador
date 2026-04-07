import json
import tkinter as tk
from tkinter import filedialog, messagebox, ttk
from collections import Counter
import cv2
import pytesseract
import re
import os
import threading
import time
import sys

def get_tesseract_path():
    if getattr(sys, 'frozen', False):
        base_path = sys._MEIPASS
    else:
        base_path = os.path.dirname(__file__)

    return os.path.join(base_path, "tesseract", "tesseract.exe")

pytesseract.pytesseract.tesseract_cmd = get_tesseract_path()

BG = "#1e1e1e"
FG = "white"
ENTRY_BG = "#2b2b2b"
BTN_BG = "#3c3f41"
BTN_ACTIVE = "#5a5a5a"

root = tk.Tk()
root.title("FILTRADOR")
root.configure(bg=BG)
root.geometry("950x700")
root.resizable(False, False)

fonte = ("Segoe UI", 10)

json_var = tk.StringVar()
pasta_var = tk.StringVar()
saida_json_var = tk.StringVar()
chave_var = tk.StringVar(value="IDPECA")
letra_var = tk.StringVar(value="C")
chaves_disponiveis = []

root.grid_rowconfigure(8, weight=1)
root.grid_columnconfigure(1, weight=1)

def normalizar(valor):
    if not valor:
        return ""
    valor = str(valor).strip().upper()
    valor = re.sub(r'[^A-Z0-9]', '', valor)
    return valor

def selecionar_json():
    caminho = filedialog.askopenfilename(filetypes=[("JSON files", "*.json")])
    if caminho:
        json_var.set(caminho)
        carregar_chaves(caminho)

def selecionar_pasta():
    caminho = filedialog.askdirectory()
    if caminho:
        pasta_var.set(caminho)

def salvar_json():
    caminho = filedialog.asksaveasfilename(defaultextension=".json")
    if caminho:
        saida_json_var.set(caminho)

def coletar_chaves(obj, chaves=None):
    if chaves is None:
        chaves = set()
    if isinstance(obj, dict):
        for k, v in obj.items():
            chaves.add(k)
            coletar_chaves(v, chaves)
    elif isinstance(obj, list):
        for item in obj:
            coletar_chaves(item, chaves)
    return chaves

def carregar_chaves(caminho_json):
    global chaves_disponiveis
    try:
        with open(caminho_json, 'r', encoding='utf-8') as f:
            dados = json.load(f)
        chaves_disponiveis = sorted(coletar_chaves(dados))
        combo_chaves['values'] = chaves_disponiveis
        if "IDPECA" in chaves_disponiveis:
            chave_var.set("IDPECA")
        elif chaves_disponiveis:
            chave_var.set(chaves_disponiveis[0])
    except Exception as e:
        messagebox.showerror("Erro", f"Erro ao carregar JSON:\n{e}")

def detectar_barcode_opencv(img):
    try:
        detector = cv2.barcode_BarcodeDetector()
        ok, decoded_info, _, _ = detector.detectAndDecode(img)
        if ok:
            return [normalizar(x) for x in decoded_info if x]
    except:
        pass
    return []

def extrair_texto_ocr(img):
    try:
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        _, thresh = cv2.threshold(gray, 150, 255, cv2.THRESH_BINARY)

        texto = pytesseract.image_to_string(thresh, config='--psm 6')

        texto_unico = texto.replace("\n", " ")

        partes = texto_unico.split()

        combinados = []
        i = 0
        while i < len(partes):
            atual = partes[i]

            if len(atual) == 1 and atual.isalpha():
                if i + 1 < len(partes) and partes[i + 1].isdigit():
                    combinados.append(atual + partes[i + 1])
                    i += 2
                    continue

            combinados.append(atual)
            i += 1

        return [normalizar(c) for c in combinados if c.strip()]

    except:
        return []

def extrair_ids_imagens(lista_caminhos, letras):
    ids = []
    padrao = re.compile(rf"^[{''.join(letras)}]\d{{8,}}$")
    for caminho in lista_caminhos:
        try:
            img = cv2.imread(caminho)
            codigos = detectar_barcode_opencv(img)
            if not codigos:
                codigos = extrair_texto_ocr(img)
            codigos = [c for c in codigos if padrao.match(c)]
            ids.extend(codigos)
        except Exception as e:
            print(f"Erro na imagem {caminho}: {e}")
    return ids

def coletar_valores(obj, chave):
    chave_normalizada = normalizar(chave)
    if isinstance(obj, dict):
        for k, v in obj.items():
            if normalizar(k) == chave_normalizada:
                yield normalizar(v)
            if isinstance(v, (dict, list)):
                yield from coletar_valores(v, chave)
    elif isinstance(obj, list):
        for item in obj:
            if isinstance(item, (dict, list)):
                yield from coletar_valores(item, chave)

def filtrar_json(obj, chave, valores_validos):
    if isinstance(obj, dict):
        for k, v in obj.items():
            if k.upper() == chave.upper() and normalizar(v) in valores_validos:
                return obj
        novo = {}
        for k, v in obj.items():
            filtrado = filtrar_json(v, chave, valores_validos)
            if filtrado:
                novo[k] = filtrado
        return novo if novo else None
    elif isinstance(obj, list):
        nova_lista = [filtrar_json(i, chave, valores_validos) for i in obj]
        nova_lista = [i for i in nova_lista if i]
        return nova_lista if nova_lista else None
    return None

def animar_label(label):
    texto = "CASA SILVA"
    pos = 0
    direcao = 1
    deslocamento_inicial = 5
    while getattr(label, "animando", False):
        display = " " * (pos + deslocamento_inicial) + texto
        label.config(text=display, fg="#006400")
        pos += direcao
        if pos > 5 or pos < 0:
            direcao *= -1
        time.sleep(0.1)
    label.config(text="")
    while getattr(label, "animando", False):
        display = " " * pos + texto
        label.config(text=display, fg="#006400")
        pos += direcao
        if pos > 0 or pos < 200:
            direcao *= -1
        time.sleep(0.1)

def processar_thread():
    arquivo_json = json_var.get()
    pasta = pasta_var.get()
    saida_json = saida_json_var.get()
    chave = chave_var.get().strip()
    letra = letra_var.get().upper()

    if not arquivo_json or not pasta:
        messagebox.showwarning("Aviso", "Selecione JSON e pasta de imagens.")
        return

    anim_label.animando = True
    anim_thread = threading.Thread(target=animar_label, args=(anim_label,), daemon=True)
    anim_thread.start()

    try:
        lista_imagens = [os.path.join(pasta, f) for f in os.listdir(pasta)
                         if f.lower().endswith((".png", ".jpg", ".jpeg", ".bmp"))]

        ids_txt = extrair_ids_imagens(lista_imagens, letra)
        ids_unicos_txt = set(ids_txt)

        with open(arquivo_json, 'r', encoding='utf-8') as f:
            dados = json.load(f)

    except Exception as e:
        anim_label.animando = False
        messagebox.showerror("Erro", f"Erro:\n{e}")
        return

    anim_label.animando = False

    valores_json = list(coletar_valores(dados, chave))
    ids_json = set(valores_json)

    validos = ids_unicos_txt & ids_json
    invalidos = ids_unicos_txt - ids_json

    resultado_text.delete(1.0, tk.END)
    resultado_text.insert(tk.END, "===== RESUMO =====\n")
    resultado_text.insert(tk.END, f"Campo: {chave}\n")
    resultado_text.insert(tk.END, f"Letra usada: {letra}\n")
    resultado_text.insert(tk.END, f"Total IDs válidos: {len(ids_txt)}\n")
    resultado_text.insert(tk.END, f"Únicos: {len(ids_unicos_txt)}\n")
    resultado_text.insert(tk.END, f"Encontrados: {len(validos)}\n")
    resultado_text.insert(tk.END, f"Não encontrados: {len(invalidos)}\n")
    if invalidos:
        resultado_text.insert(tk.END, "\nNão encontrados:\n")
        for cod in sorted(invalidos):
            resultado_text.insert(tk.END, f"{cod}\n")

    dados_filtrados = filtrar_json(dados, chave, ids_unicos_txt)
    if saida_json and dados_filtrados:
        with open(saida_json, 'w', encoding='utf-8') as f:
            json.dump(dados_filtrados, f, indent=4, ensure_ascii=False)
        resultado_text.insert(tk.END, f"\n\nJSON salvo em:\n{saida_json}")

def processar():
    threading.Thread(target=processar_thread).start()

def label(texto, row):
    tk.Label(root, text=texto, bg=BG, fg=FG, font=fonte)\
        .grid(row=row, column=0, sticky="e", padx=5, pady=5)

def entry(var, row):
    tk.Entry(root, textvariable=var, width=60,
             bg=ENTRY_BG, fg="black", insertbackground="black", font=fonte,
             state="readonly")\
        .grid(row=row, column=1, padx=5, pady=5)

def button(texto, comando, row, col):
    tk.Button(root, text=texto, command=comando,
              bg=BTN_BG, fg=FG, activebackground=BTN_ACTIVE,
              font=fonte)\
        .grid(row=row, column=col, padx=5, pady=5)

label("Arquivo JSON:", 0)
entry(json_var, 0)
button("Selecionar", selecionar_json, 0, 2)

label("Pasta de Imagens:", 1)
entry(pasta_var, 1)
button("Selecionar", selecionar_pasta, 1, 2)

label("Salvar JSON:", 2)
entry(saida_json_var, 2)
button("Salvar como", salvar_json, 2, 2)

label("Campo filtro:", 3)
combo_chaves = ttk.Combobox(root, textvariable=chave_var, values=chaves_disponiveis, width=57, state="readonly")
combo_chaves.grid(row=3, column=1, padx=5, pady=5)

label("Tipo de código:", 4)
combo_letras = ttk.Combobox(root, textvariable=letra_var, values=["A","B","C","G","P"], width=5, state="readonly")
combo_letras.grid(row=4, column=1, padx=5, pady=5)

tk.Button(root, text="Executar", command=processar, bg="#4CAF50", fg="white",
          font=("Segoe UI", 11, "bold")).grid(row=5, column=0, columnspan=3, padx=(30, 0), pady=10)

anim_label = tk.Label(root, text="", bg=BG, fg="cyan",
                      font=("Segoe UI", 14, "bold"), anchor="center")
anim_label.grid(row=6, column=0, columnspan=3, sticky="nsew", pady=10)

resultado_text = tk.Text(root, bg=BG, fg=FG, insertbackground=FG,
                         font=("Consolas", 10), wrap="word")
resultado_text.grid(row=8, column=0, columnspan=3, padx=10, pady=10, sticky="nsew")

root.mainloop()
