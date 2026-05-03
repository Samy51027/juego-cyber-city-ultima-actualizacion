# juego-cyber-city-ultima-actualizacion

import tkinter as tk
from tkinter import font as tkfont
from PIL import Image, ImageTk, ImageEnhance, ImageDraw
import os, sys, random

#el archivo se abre mientreas este en una ubicacion especifica
os.chdir(os.path.dirname(os.path.abspath(__file__)))


#  Define el tamaño que se abrira la ventana

ANCHO = 960
ALTO  = 640

# C, especifica los colores de las letrar

C = {
    "bg"      : "#0A0514",
    "cyan"    : "#00FFC8",
    "magenta" : "#FF3CFF",
    "amarillo": "#FFDC00",
    "rojo"    : "#FF1E50",
    "gris"    : "#787890",
    "verde"   : "#32FF64",
    "blanco"  : "#FFFFFF",
    "panel"   : "#0F0520",
    "card_bg" : "#0A0416",
    "card_sel": "#231040",
    "azul"    : "#3C96FF",
}


# Diccionarios de los entrenadores

ENTRENADORES = [
    {"nombre": "REX",   "avatar": "rex.png",   "color": "#DC5050", "desc": "Agresivo y rapido"},
    {"nombre": "BEN",   "avatar": "ben.png",   "color": "#508CDC", "desc": "Cauteloso y analitico"},
    {"nombre": "MARIA", "avatar": "maria.png", "color": "#C850C8", "desc": "Equilibrada y firme"},
]


# listas paralelas de los distritos

UBICACIONES = ["DISTRITO NEON", "PUERTO CYBER", "ZONA GLITCH", "TORRE CORE", "NEXO FINAL"]
COL_UBI     = ["#5000C8", "#0096C8", "#C83232", "#32C864", "#C89600"]


#  VENTANA

root = tk.Tk()
root.title("CYBER CITY")
root.geometry(f"{ANCHO}x{ALTO}")
root.resizable(False, False)
root.configure(bg=C["bg"])

#letras, tipo, tamaño y negrita

try:
    F_TITULO = tkfont.Font(family="Press Start 2P", size=26, weight="bold")
    F_MED    = tkfont.Font(family="Press Start 2P", size=11)
    F_SMALL  = tkfont.Font(family="Press Start 2P", size=9)
    F_TINY   = tkfont.Font(family="Press Start 2P", size=7)
except:
    F_TITULO = tkfont.Font(family="Arial", size=30, weight="bold")
    F_MED    = tkfont.Font(family="Arial", size=14, weight="bold")
    F_SMALL  = tkfont.Font(family="Arial", size=11, weight="bold")
    F_TINY   = tkfont.Font(family="Arial", size=9)


#CANVAS

canvas = tk.Canvas(root, width=ANCHO, height=ALTO,
                   bg=C["bg"], highlightthickness=0)
canvas.pack(fill="both", expand=True)


# Guardar la informacion del usurio jugador
estado = {
    "nombre"    : "",
    "entrenador": 0,
    "equipo"    : [],
    "ubicacion" : 0,
    "victorias" : 0,
    "puntaje"   : 0,
}

_after_ids = []


#  Cache de imagenes

_cache_img = {}

def buscar_img(nombre):
    """Dada una cadena (puede ser ruta ya resuelta o solo nombre),
    devuelve la primera ruta existente o la cadena original."""
    if not nombre:
        return ""
    nombre = nombre.strip()
    # Si ya existe tal cual, devolverla directamente
    if os.path.exists(nombre):
        return nombre
    base = os.path.splitext(nombre)[0]
    extensiones = [".png", ".PNG", ".jpg", ".JPG", ".jpeg", ".JPEG"]
    carpetas    = ["", "personajes", "imagenes", "imgs", "assets", "sprites"]


#Genera todas las combinaciones posibles de png, jpg...
    def construir_rutas(ci, ei, acc):
        if ci >= len(carpetas):
            return acc
        if ei >= len(extensiones):
            return construir_rutas(ci + 1, 0, acc)
        carpeta = carpetas[ci]
        ruta = (base + extensiones[ei]) if carpeta == "" \
               else os.path.join(carpeta, base + extensiones[ei])
        return construir_rutas(ci, ei + 1, acc + [ruta])
 
 # Devuelve la ruta que se encontro
    def probar_ruta(rutas, idx):
        if idx >= len(rutas):
            return nombre
        if os.path.exists(rutas[idx]):
            return rutas[idx]
        return probar_ruta(rutas, idx + 1)

    return probar_ruta(construir_rutas(0, 0, []), 0)

def cargar_img(ruta, tam, flip=False):
    key = (ruta, tam, flip)
    if key in _cache_img:
        return _cache_img[key]
    try:
        img = Image.open(ruta).convert("RGBA")
        if flip:
            img = img.transpose(Image.FLIP_LEFT_RIGHT)
        img    = img.resize(tam, Image.LANCZOS)
        tk_img = ImageTk.PhotoImage(img)
        _cache_img[key] = tk_img
        return tk_img
    except:
        w, h = tam
        img  = Image.new("RGBA", (w, h), (40, 10, 70, 255))
        draw = ImageDraw.Draw(img)
        draw.rectangle([0, 0, w-1, h-1], outline=(120, 40, 180), width=2)
        letra = ruta[0].upper() if ruta else "?"
        draw.text((w//2-6, h//2-8), letra, fill=(0, 255, 200))
        tk_img = ImageTk.PhotoImage(img)
        _cache_img[key] = tk_img
        return tk_img

def cargar_fondo(nombre):
    key = ("fondo", nombre)
    if key in _cache_img:
        return _cache_img[key]
    ruta = buscar_img(nombre)
    try:
        img    = Image.open(ruta).convert("RGBA")
        img    = img.resize((ANCHO, ALTO), Image.LANCZOS)
        img    = ImageEnhance.Brightness(img).enhance(0.5)
        tk_img = ImageTk.PhotoImage(img)
    except:
        img    = Image.new("RGBA", (ANCHO, ALTO), (10, 5, 20, 255))
        tk_img = ImageTk.PhotoImage(img)
    _cache_img[key] = tk_img
    return tk_img


#  LEER PERSONAJES 

def leer_personajes():
    try:
        with open("datos_personajes.txt", "r", encoding="utf-8") as f:
            lineas = f.readlines()
    except Exception as ex:
        print(f"Error: {ex}")
        return []

# posición actual en la lista= idx   ,  lista donde guardas personajes= acc

    def procesar(idx, acc):
        if idx >= len(lineas):       
            return acc
        linea = lineas[idx].strip()  #quita espacios de la linea
        if not linea:
            return procesar(idx + 1, acc)  #Saltar líneas vacías
        partes = linea.split(",")          #Separar datos para luego añadirles comillas
        if len(partes) < 5:                #Si la línea está mal la ignora
            return procesar(idx + 1, acc)
        try:
            nombre = partes[0].strip()              #Convierte texto a números
            vida   = int(partes[1].strip())
            atk    = int(partes[2].strip())
            def_   = int(partes[3].strip())
            av_raw = partes[4].strip().split()[0] if partes[4].strip() else ""  #Busca la ruta real de la imagen

            avatar = buscar_img(av_raw)
            acc.append({"nombre": nombre, "vida_max": vida, "vida": vida,        #Guarda cada personaje como diccionario
                        "atk": atk, "def": def_, "avatar": avatar})
            print(f"  OK: {nombre} | {avatar} | existe={os.path.exists(avatar)}")   #muestra si la imagen existe
        except Exception as ex:
            print(f"  Error linea {idx}: {ex}")
        return procesar(idx + 1, acc)                #Va a la siguiente línea

    return procesar(1, [])                           #Empieza desde la línea 1 

TODOS_PERSONAJES = leer_personajes()
print(f"Total personajes: {len(TODOS_PERSONAJES)}")

FB_NOMBRE     = cargar_fondo("fondo_nombre.jpg")
FB_ENTRENADOR = cargar_fondo("fondo_entrenadores.jpg")
FB_PERSONAJES = cargar_fondo("fondo_personajes.jpg")

def get_fondo_batalla(idx):
    return cargar_fondo(f"fondo_batalla_{idx}.jpg")


#  UTILIDADES UI

def cancelar_afters():
    def cancel(idx):
        if idx >= len(_after_ids):
            return
        try:
            root.after_cancel(_after_ids[idx])
        except:
            pass
        cancel(idx + 1)
    cancel(0)
    _after_ids.clear()

#reinicia la interfaz
def limpiar():
    cancelar_afters()
    canvas.delete("all")
    root.unbind("<KeyPress>")
    def destruir(widgets, idx):
        if idx >= len(widgets):
            return
        try:
            widgets[idx].destroy()
        except:
            pass
        destruir(widgets, idx + 1)
    destruir(canvas.winfo_children(), 0)

def dibujar_fondo(img_tk):
    canvas.create_image(0, 0, anchor="nw", image=img_tk)
    canvas.create_rectangle(0, 0, ANCHO, ALTO,
                             fill="#050210", stipple="gray50")

def texto_sombra(x, y, txt, fuente, color,
                 sombra="#000000", anchor="center"):
    canvas.create_text(x+2, y+2, text=txt, font=fuente,
                       fill=sombra, anchor=anchor)
    canvas.create_text(x,   y,   text=txt, font=fuente,
                       fill=color,  anchor=anchor)

def boton_cv(x, y, w, h, txt, borde, col_txt, fuente, cmd, tag):    #crea botones
    x1, y1 = x - w//2, y - h//2
    x2, y2 = x + w//2, y + h//2
    r = canvas.create_rectangle(x1, y1, x2, y2, fill=C["panel"],
                                  outline=borde, width=2, tags=tag)
    t = canvas.create_text(x, y, text=txt, font=fuente,
                            fill=col_txt, tags=tag)
    canvas.tag_bind(tag, "<Enter>",                                   #Cambia color cuando pasa el mouse
        lambda e: canvas.itemconfig(r, outline=C["cyan"], width=3))
    canvas.tag_bind(tag, "<Leave>",
        lambda e: canvas.itemconfig(r, outline=borde, width=2))
    canvas.tag_bind(tag, "<Button-1>", lambda e: cmd())               #Ejecuta función al hacer click
    return r, t

def barra_vida_cv(x, y, w, h, actual, maximo, color):
    canvas.create_rectangle(x, y, x+w, y+h,
                             fill="#281428", outline=C["gris"])
    if maximo > 0 and actual > 0:
        llen = int(w * actual / maximo)
        canvas.create_rectangle(x, y, x+llen, y+h,
                                 fill=color, outline="")
    canvas.create_rectangle(x, y, x+w, y+h,
                             fill="", outline=C["gris"])

def borrar_ids(its, i=0):
    if i >= len(its):
        return
    try:
        canvas.delete(its[i])
    except:
        pass
    borrar_ids(its, i + 1)


#  ABOUT          #función a la que regresa cuando presiona “VOLVER”

def pantalla_about(volver_cb):
    limpiar()
    dibujar_fondo(FB_NOMBRE)
    texto_sombra(ANCHO//2, 80, "ACERCA", F_TITULO, C["cyan"])

    lineas = [
        ("CYBER CITY - Juego de Batallas", C["amarillo"]),
        ("", C["bg"]),
        ("Juego de estrategia por turnos.", C["blanco"]),
        ("Elige 3 personajes y enfrenta", C["blanco"]),
        ("al Hollow en 5 ubicaciones.", C["blanco"]),
        ("", C["bg"]),
        ("Mecanica: ATK - DEF = Dano ", C["cyan"]),
        ("DEFENDER: reduce dano recibido 50%", C["azul"]),
        ("", C["bg"]),
        ("El Hollow actua aleatoriamente.", C["blanco"]),
        ("Captura sus personajes para ganar.", C["blanco"]),
        ("", C["bg"]),
        ("Proyecto de intro - 2026", C["gris"]),
    ]

    def dibujar_lineas(idx, y):
        if idx >= len(lineas):
            return
        txt, col = lineas[idx]
        if txt:
            canvas.create_text(ANCHO//2, y, text=txt,
                               font=F_TINY, fill=col)
        dibujar_lineas(idx + 1, y + 28)

    dibujar_lineas(0, 160)
    boton_cv(ANCHO//2, 560, 200, 40, "< VOLVER",
             C["magenta"], C["magenta"], F_SMALL, volver_cb, "btn_v")


#  PASO 1 — NOMBRE

_gif_frames = []
_gif_idx    = [0]

def cargar_gif():
    if _gif_frames:
        return
    try:
        gif = Image.open("fondo.gif")
        def acumular(i, acc):
            try:
                gif.seek(i)
                fr = gif.convert("RGBA").resize((ANCHO, ALTO), Image.LANCZOS)
                fr = ImageEnhance.Brightness(fr).enhance(0.45)
                acc.append(ImageTk.PhotoImage(fr))
                return acumular(i + 1, acc)
            except EOFError:
                return acc
        _gif_frames.extend(acumular(0, []))
        print(f"GIF: {len(_gif_frames)} frames")
    except Exception as ex:
        print(f"Sin GIF: {ex}")

def pantalla_nombre():
    limpiar()
    cargar_gif()

    nombre_var = tk.StringVar()              #Guarda lo que escribe el usuario
    bg_item    = [None]                      #Guarda el fondo animado

    def animar():
        if _gif_frames:
            if bg_item[0]:
                canvas.delete(bg_item[0])
            bg_item[0] = canvas.create_image(                          #Dibuja frame actual
                0, 0, anchor="nw", image=_gif_frames[_gif_idx[0]])
            canvas.tag_lower(bg_item[0])
            _gif_idx[0] = (_gif_idx[0] + 1) % len(_gif_frames)          #Avanza al siguiente frame
            canvas.create_rectangle(0, 0, ANCHO, ALTO,
                                     fill="#050210", stipple="gray50",
                                     tags="overlay")
            canvas.tag_lower("overlay")
        aid = root.after(80, animar)             #Repite cada 80 ms
        _after_ids.append(aid)

    if not _gif_frames:
        dibujar_fondo(FB_NOMBRE)

    canvas.create_text(ANCHO//2+2, 112, text="CYBER CITY",
                       font=F_TITULO, fill="#500050")
    canvas.create_text(ANCHO//2, 110, text="CYBER CITY",
                       font=F_TITULO, fill=C["magenta"])
    canvas.create_text(ANCHO//2, 170,
                       text="Un juego de batallas por turnos",
                       font=F_SMALL, fill=C["cyan"])
    canvas.create_text(ANCHO//2, 252,
                       text="INGRESA TU NOMBRE:",
                       font=F_SMALL, fill=C["cyan"])
    canvas.create_text(ANCHO//2, 540,
                       text="Presiona ENTER para continuar",
                       font=F_TINY, fill=C["gris"])

    err_id = canvas.create_text(ANCHO//2, 346, text="",
                                 font=F_TINY, fill=C["rojo"])

    entry = tk.Entry(root, textvariable=nombre_var, font=F_MED,
                     bg="#0F0520", fg=C["blanco"],
                     insertbackground=C["cyan"],
                     relief="flat", justify="center",
                     highlightthickness=2,
                     highlightcolor=C["cyan"],
                     highlightbackground=C["magenta"])
    canvas.create_window(ANCHO//2, 296, window=entry, width=360, height=46)
    entry.focus_set()

    def intentar_continuar():
        n = nombre_var.get().strip()
        if not n:
            canvas.itemconfig(err_id, text="Debes ingresar tu nombre!")
            return
        estado["nombre"] = n
        pantalla_entrenador()

    entry.bind("<Return>", lambda e: intentar_continuar())

    boton_cv(ANCHO//2 - 140, 476, 220, 44, "ABOUT",
             C["gris"], C["gris"], F_SMALL,
             lambda: pantalla_about(pantalla_nombre), "btn_about")
    boton_cv(ANCHO//2 + 140, 476, 220, 44, "INICIAR >",
             C["magenta"], C["magenta"], F_SMALL,
             intentar_continuar, "btn_ini")

    if _gif_frames:
        animar()


#  PASO 2 — ENTRENADOR

def pantalla_entrenador():
    limpiar()
    dibujar_fondo(FB_ENTRENADOR)

    texto_sombra(ANCHO//2, 44, "ELIGE TU ENTRENADOR", F_MED, C["cyan"])
    canvas.create_text(ANCHO//2, 70,
                       text=f"Jugador: {estado['nombre']}",
                       font=F_TINY, fill=C["amarillo"])

    sel      = [estado.get("entrenador", 0)]
    imgs_ent = {}
    card_ids = [[], [], []]

    card_w, card_h = 220, 340
    gap    = 24
    total  = 3 * card_w + 2 * gap
    sx     = (ANCHO - total) // 2
    cy_top = 96

    def precargar_ents(idx):
        if idx >= len(ENTRENADORES):
            return
        ent = ENTRENADORES[idx]
        imgs_ent[idx] = cargar_img(buscar_img(ent["avatar"]), (168, 210))
        precargar_ents(idx + 1)

    precargar_ents(0)

    def borrar_card(idx):
        def _b(its, i):
            if i >= len(its):
                return
            try:
                canvas.delete(its[i])
            except:
                pass
            _b(its, i+1)
        _b(card_ids[idx], 0)
        card_ids[idx].clear()

    def dibujar_card(idx):
        borrar_card(idx)
        ent    = ENTRENADORES[idx]
        cx     = sx + idx * (card_w + gap)
        es_sel = (idx == sel[0])
        ids    = card_ids[idx]

        bg     = C["card_sel"] if es_sel else C["card_bg"]
        borde  = ent["color"] if es_sel else "#321450"
        grosor = 3 if es_sel else 1

        r = canvas.create_rectangle(cx, cy_top, cx+card_w, cy_top+card_h,
                                     fill=bg, outline=borde, width=grosor)
        ids.append(r)

        if idx in imgs_ent:
            ii = canvas.create_image(cx+card_w//2, cy_top+118,
                                      anchor="center", image=imgs_ent[idx])
            ids.append(ii)

        nm = canvas.create_text(cx+card_w//2, cy_top+232,
                                  text=ent["nombre"], font=F_SMALL,
                                  fill=ent["color"] if es_sel else C["gris"])
        ids.append(nm)

        sep = canvas.create_line(cx+16, cy_top+245, cx+card_w-16, cy_top+245,
                                   fill=ent["color"] if es_sel else "#321450")
        ids.append(sep)

        dc = canvas.create_text(cx+card_w//2, cy_top+264,
                                  text=ent["desc"], font=F_TINY,
                                  fill=C["cyan"] if es_sel else "#404060")
        ids.append(dc)

        if es_sel:
            bx1 = cx+12;        bx2 = cx+card_w-12
            by1 = cy_top+card_h-40; by2 = cy_top+card_h-10
            bg_ = canvas.create_rectangle(bx1, by1, bx2, by2,
                                           fill=ent["color"], outline="")
            bt_ = canvas.create_text((bx1+bx2)//2, (by1+by2)//2,
                                      text="SELECCIONADO",
                                      font=F_TINY, fill="#000000")
            ids.append(bg_)
            ids.append(bt_)

        def hacer_click(i):
            def _c(e):
                if sel[0] == i:
                    confirmar()
                else:
                    sel[0] = i
                    redibujar_todas()
            return _c

        def bind_ids(its, i, fn):
            if i >= len(its):
                return
            canvas.tag_bind(its[i], "<Button-1>", fn)
            bind_ids(its, i+1, fn)

        bind_ids(ids, 0, hacer_click(idx))

    def redibujar_todas():
        dibujar_card(0); dibujar_card(1); dibujar_card(2)
        actualizar_btn_ok()

    def confirmar():
        estado["entrenador"] = sel[0]
        pantalla_personajes()

    def volver():
        estado["entrenador"] = sel[0]
        pantalla_nombre()

    dibujar_card(0); dibujar_card(1); dibujar_card(2)

    canvas.create_text(ANCHO//2, 458,
                       text="Clic para elegir | Flechas | ENTER para confirmar",
                       font=F_TINY, fill=C["gris"])

    boton_cv(90, 550, 150, 38, "< VOLVER",
             C["gris"], C["gris"], F_TINY, volver, "btn_v")

    btn_ok_ids = [None, None]

    def actualizar_btn_ok():
        if btn_ok_ids[0]:
            canvas.delete(btn_ok_ids[0])
        if btn_ok_ids[1]:
            canvas.delete(btn_ok_ids[1])
        n = ENTRENADORES[sel[0]]["nombre"]
        r, t = boton_cv(ANCHO-185, 550, 330, 38,
                         f"ELEGIR A {n} >",
                         C["magenta"], C["magenta"], F_TINY,
                         confirmar, "btn_ok")
        btn_ok_ids[0] = r
        btn_ok_ids[1] = t

    actualizar_btn_ok()

    def tecla(e):
        if e.keysym == "Left":
            sel[0] = (sel[0]-1) % 3
            redibujar_todas()
        elif e.keysym == "Right":
            sel[0] = (sel[0]+1) % 3
            redibujar_todas()
        elif e.keysym == "Return":
            confirmar()
        elif e.keysym == "Escape":
            volver()

    root.bind("<KeyPress>", tecla)

# ══════════════════════════════════════════════════════════
#  PASO 3 — PERSONAJES
# ══════════════════════════════════════════════════════════
def pantalla_personajes():
    limpiar()
    dibujar_fondo(FB_PERSONAJES)

    ent       = ENTRENADORES[estado["entrenador"]]
    sel       = []
    imgs_mini = {}
    card_refs = []
    prev_ids  = []
    av_sel    = [0]
    av_ids    = []
    error_id  = [None]
    inst_id   = [None]

    img_ent_m = cargar_img(buscar_img(ent["avatar"]), (52, 64))
    canvas.create_image(ANCHO-60, 8, anchor="nw", image=img_ent_m)
    canvas.create_text(ANCHO-66, 76, text=ent["nombre"],
                       font=F_TINY, fill=ent["color"], anchor="e")

    texto_sombra(ANCHO//2, 26, "ELIGE TU EQUIPO", F_MED, C["cyan"])
    canvas.create_text(ANCHO//2, 48,
                       text=f"Jugador: {estado['nombre']}",
                       font=F_TINY, fill=C["amarillo"])

    inst_id[0] = canvas.create_text(
        16, 64,
        text="Selecciona 3 personajes (0/3):",
        font=F_TINY, fill=C["magenta"], anchor="w")

    # ── CORRECCIÓN: p["avatar"] ya es la ruta resuelta, cargar directo ──
    def precargar(idx):
        if idx >= len(TODOS_PERSONAJES):
            return
        p = TODOS_PERSONAJES[idx]
        imgs_mini[idx] = cargar_img(p["avatar"], (68, 68))
        precargar(idx + 1)

    precargar(0)

    COLS   = 5
    cw, ch = 170, 120
    gx, gy = 6, 5
    tw_g   = COLS * cw + (COLS-1) * gx
    sx_g   = (ANCHO - tw_g) // 2
    sy_g   = 76

    def dibujar_cards(idx):
        if idx >= len(TODOS_PERSONAJES):
            return
        p    = TODOS_PERSONAJES[idx]
        col  = idx % COLS
        fila = idx // COLS
        cx   = sx_g + col * (cw + gx)
        cy   = sy_g + fila * (ch + gy)

        es_sel = idx in sel
        bg     = C["card_sel"] if es_sel else C["card_bg"]
        borde  = C["cyan"] if es_sel else "#321450"
        grosor = 2 if es_sel else 1

        ids = []
        r   = canvas.create_rectangle(cx, cy, cx+cw, cy+ch,
                                        fill=bg, outline=borde, width=grosor)
        ids.append(r)

        if idx in imgs_mini and imgs_mini[idx]:
            ii = canvas.create_image(cx+cw//2, cy+42,
                                      anchor="center", image=imgs_mini[idx])
            ids.append(ii)

        nm_c = C["cyan"] if es_sel else C["blanco"]
        nm   = canvas.create_text(cx+cw//2, cy+ch-30,
                                   text=p["nombre"][:10],
                                   font=F_TINY, fill=nm_c)
        ids.append(nm)

        st = canvas.create_text(cx+cw//2, cy+ch-14,
                                  text=f"HP:{p['vida_max']} A:{p['atk']} D:{p['def']}",
                                  font=F_TINY, fill=C["gris"])
        ids.append(st)

        if es_sel:
            num = sel.index(idx) + 1
            bc  = canvas.create_oval(cx+cw-22, cy+4, cx+cw-2, cy+24,
                                      fill=C["cyan"], outline="")
            bt  = canvas.create_text(cx+cw-12, cy+14, text=str(num),
                                      font=F_TINY, fill="#000000")
            ids.append(bc)
            ids.append(bt)

        def hacer_click(i):
            def _c(e):
                if i in sel:
                    sel.remove(i)
                elif len(sel) < 4:
                    sel.append(i)
                canvas.itemconfig(error_id[0], text="")
                canvas.itemconfig(
                    inst_id[0],
                    text=f"Selecciona 4 personajes ({len(sel)}/4):")
                redibujar_cards()
                redibujar_preview()
            return _c

        def bind_ids_card(its, i, fn):
            if i >= len(its):
                return
            canvas.tag_bind(its[i], "<Button-1>", fn)
            bind_ids_card(its, i+1, fn)

        bind_ids_card(ids, 0, hacer_click(idx))
        card_refs.append(ids)
        dibujar_cards(idx + 1)

    def redibujar_cards():
        def borrar_all(refs, i):
            if i >= len(refs):
                return
            borrar_ids(refs[i])
            borrar_all(refs, i+1)
        borrar_all(card_refs, 0)
        card_refs.clear()
        dibujar_cards(0)

    dibujar_cards(0)

    AV_COLS = ["#C850C8", "#50C8C8", "#C8C850"]
    py_av   = 418

    canvas.create_text(16, py_av, text="ELIGE TU AVATAR:",
                       font=F_TINY, fill=C["cyan"], anchor="w")

    def dibujar_avatares(idx):
        if idx >= 3:
            return
        ax = 16 + idx * 62
        ay = py_av + 18
        es = (idx == av_sel[0])
        r  = canvas.create_rectangle(ax, ay, ax+52, ay+52,
                                      fill=AV_COLS[idx],
                                      outline=C["amarillo"] if es else "#3C1450",
                                      width=3 if es else 1)
        t  = canvas.create_text(ax+26, ay+26, text=str(idx+1),
                                  font=F_SMALL, fill="#000000")
        av_ids.append(r)
        av_ids.append(t)

        def sel_av(i):
            def _c(e):
                av_sel[0] = i
                borrar_ids(av_ids)
                av_ids.clear()
                dibujar_avatares(0)
            return _c

        canvas.tag_bind(r, "<Button-1>", sel_av(idx))
        canvas.tag_bind(t, "<Button-1>", sel_av(idx))
        dibujar_avatares(idx + 1)

    dibujar_avatares(0)

    def redibujar_preview():
        borrar_ids(prev_ids)
        prev_ids.clear()
        t = canvas.create_text(ANCHO//2+20, py_av,
                               text="EQUIPO:", font=F_TINY,
                               fill=C["amarillo"], anchor="w")
        prev_ids.append(t)

        def dibujar_miembro(idx):
            if idx >= len(sel):
                return
            p_s  = TODOS_PERSONAJES[sel[idx]]
            # ── CORRECCIÓN: usar p_s["avatar"] directamente ──
            mini = cargar_img(p_s["avatar"], (44, 44))
            mx   = ANCHO//2 + 20 + idx * 56
            my   = py_av + 16
            ii   = canvas.create_image(mx, my, anchor="nw", image=mini)
            nm   = canvas.create_text(mx+22, my+48,
                                       text=p_s["nombre"][:6],
                                       font=F_TINY, fill=C["cyan"])
            prev_ids.append(ii)
            prev_ids.append(nm)
            dibujar_miembro(idx + 1)

        dibujar_miembro(0)

    redibujar_preview()

    error_id[0] = canvas.create_text(ANCHO//2, 484, text="",
                                      font=F_SMALL, fill=C["rojo"])

    def confirmar():
        if len(sel) < 3:
            canvas.itemconfig(error_id[0],
                text=f"Selecciona 3 personajes! ({len(sel)}/3)")
            return
        equipo = []
        def armar(idx):
            if idx >= len(sel):
                return
            p = dict(TODOS_PERSONAJES[sel[idx]])
            p["vida"] = p["vida_max"]
            equipo.append(p)
            armar(idx + 1)
        armar(0)
        estado["equipo"] = equipo
        pantalla_mapa()

    boton_cv(90, 558, 160, 38, "< VOLVER",
             C["gris"], C["gris"], F_TINY,
             pantalla_entrenador, "btn_v")
    boton_cv(ANCHO-170, 558, 300, 38, "CONFIRMAR EQUIPO >",
             C["magenta"], C["magenta"], F_TINY, confirmar, "btn_ok")


#  MAPA
# 
def pantalla_mapa():
    limpiar()
    dibujar_fondo(FB_NOMBRE)

    ent       = ENTRENADORES[estado["entrenador"]]
    ubi_act   = estado["ubicacion"]
    victorias = estado["victorias"]

    texto_sombra(ANCHO//2, 38, "MAPA CYBER CITY", F_MED, C["cyan"])
    canvas.create_text(
        ANCHO//2, 64,
        text=f"JUGADOR: {estado['nombre']}   VICTORIAS: {victorias}/5",
        font=F_TINY, fill=C["amarillo"])

    img_ent_m = cargar_img(buscar_img(ent["avatar"]), (58, 72))
    canvas.create_image(16, ALTO-88, anchor="nw", image=img_ent_m)
    canvas.create_text(80, ALTO-50, text=ent["nombre"],
                       font=F_TINY, fill=ent["color"], anchor="w")

    nodos = []
    def calc_nodos(i):
        if i >= 5:
            return
        nodos.append((int(ANCHO*(i+1)/6), ALTO//2))
        calc_nodos(i + 1)
    calc_nodos(0)

    def dibujar_lineas(i):
        if i >= 4:
            return
        col = C["cyan"] if i < ubi_act else C["gris"]
        canvas.create_line(nodos[i][0], nodos[i][1],
                            nodos[i+1][0], nodos[i+1][1],
                            fill=col, width=3)
        dibujar_lineas(i + 1)

    dibujar_lineas(0)

    def dibujar_nodos(i):
        if i >= 5:
            return
        nx, ny  = nodos[i]
        radio   = 38
        complet = i < ubi_act
        actual  = i == ubi_act
        bloq    = i > ubi_act

        fill_c  = COL_UBI[i] if not bloq else "#281440"
        borde_c = C["amarillo"] if actual else (C["cyan"] if complet else C["gris"])
        grosor  = 4 if actual else 2

        canvas.create_oval(nx-radio, ny-radio, nx+radio, ny+radio,
                            fill=fill_c, outline=borde_c, width=grosor)

        lbl     = "OK" if complet else ("!" if actual else "?")
        col_lbl = C["verde"] if complet else (C["amarillo"] if actual else C["gris"])
        canvas.create_text(nx, ny, text=lbl, font=F_SMALL, fill=col_lbl)
        canvas.create_text(nx, ny+52, text=UBICACIONES[i],
                            font=F_TINY,
                            fill=C["blanco"] if not bloq else C["gris"])
        dibujar_nodos(i + 1)

    dibujar_nodos(0)

    if ubi_act < 5:
        boton_cv(ANCHO//2, 578, 320, 42,
                 f"IR A {UBICACIONES[ubi_act]} >",
                 C["magenta"], C["magenta"], F_MED,
                 pantalla_batalla, "btn_bat")


#  BATALLA

def pantalla_batalla():
    limpiar()

    ubi_idx = estado["ubicacion"]
    ent     = ENTRENADORES[estado["entrenador"]]
    eq_jug  = [dict(p) for p in estado["equipo"]]

    # Crear equipo hollow recursivamente
    pool = random.sample(TODOS_PERSONAJES, 3)
    def armar_hollow(idx, acc):
        if idx >= len(pool):
            return acc
        p = dict(pool[idx])
        f = 1.0 + ubi_idx * 0.1
        p["vida_max"] = int(p["vida_max"] * f)
        p["vida"]     = p["vida_max"]
        p["atk"]      = int(p["atk"] * f)
        acc.append(p)
        return armar_hollow(idx + 1, acc)
    eq_hol = armar_hollow(0, [])

    bat = {
        "idx_jug"    : 0,
        "idx_hol"    : 0,
        "pts_jug"    : estado["puntaje"],
        "pts_hol"    : 0,
        "turno"      : "jugador",
        "log"        : [],
        "fin"        : None,
        "modo"       : "accion",    # "accion" | "cambio"
        "defendiendo": False,       # True cuando el jugador eligio DEFENDER este turno
    }

    fondo_bat = get_fondo_batalla(ubi_idx)

    # ── CORRECCIÓN: usar p["avatar"] directamente (ya es ruta resuelta) ──
    imgs_bat = {}
    def precargar_bat(personajes, idx):
        if idx >= len(personajes):
            return
        p    = personajes[idx]
        ruta = p["avatar"]
        imgs_bat[(p["nombre"], False)] = cargar_img(ruta, (100, 100), flip=False)
        imgs_bat[(p["nombre"], True)]  = cargar_img(ruta, (100, 100), flip=True)
        precargar_bat(personajes, idx + 1)

    precargar_bat(eq_jug, 0)
    precargar_bat(eq_hol, 0)
    img_ent_bat = cargar_img(buscar_img(ent["avatar"]), (74, 92))

    def agregar_log(msg):
        bat["log"].append(msg)
        if len(bat["log"]) > 5:
            bat["log"].pop(0)

    def calcular_dano(atk, def_):
        return max(1, atk - def_)

    
    def primer_vivo_jug(excluir=-1):
        def buscar(idx):
            if idx >= len(eq_jug):
                return -1
            if eq_jug[idx]["vida"] > 0 and idx != excluir:
                return idx
            return buscar(idx + 1)
        return buscar(0)

    def primer_vivo_hol(excluir=-1):
        def buscar(idx):
            if idx >= len(eq_hol):
                return -1
            if eq_hol[idx]["vida"] > 0 and idx != excluir:
                return idx
            return buscar(idx + 1)
        return buscar(0)

    def hay_vivos_jug():
        return primer_vivo_jug() != -1

    def hay_vivos_hol():
        return primer_vivo_hol() != -1

    
    #  RENDER
    
    def render():
        canvas.delete("all")
        canvas.create_image(0, 0, anchor="nw", image=fondo_bat)
        canvas.create_rectangle(0, 0, ANCHO, ALTO,
                                 fill="#0A0005", stipple="gray50")

        texto_sombra(ANCHO//2, 20,
                     f">> {UBICACIONES[ubi_idx]} <<",
                     F_MED, C["amarillo"])

        # Entrenador
        canvas.create_image(8, ALTO-102, anchor="nw", image=img_ent_bat)
        canvas.create_text(88, ALTO-56, text=ent["nombre"],
                            font=F_TINY, fill=ent["color"], anchor="w")

        pj = eq_jug[bat["idx_jug"]] if bat["idx_jug"] < len(eq_jug) else None
        ph = eq_hol[bat["idx_hol"]] if bat["idx_hol"] < len(eq_hol) else None

        # Panel jugador
        canvas.create_text(20, 44,
                            text=f"{estado['nombre']}  PTS:{bat['pts_jug']}",
                            font=F_TINY, fill=C["cyan"], anchor="w")

        # Indicador de DEFENDER activo
        if bat["defendiendo"]:
            canvas.create_rectangle(12, 54, 200, 72,
                                     fill="#001840", outline=C["azul"], width=1)
            canvas.create_text(106, 63,
                                text="** DEFENDIENDO **",
                                font=F_TINY, fill=C["azul"])

        if pj:
            px, py = 170, 128
            img_pj = imgs_bat.get((pj["nombre"], False))
            if img_pj:
                canvas.create_image(px, py, anchor="center", image=img_pj)
            # Halo azul si está defendiendo
            if bat["defendiendo"]:
                canvas.create_oval(px-55, py-55, px+55, py+55,
                                    fill="", outline=C["azul"], width=3)
            canvas.create_text(px, py+62, text=pj["nombre"],
                                font=F_SMALL, fill=C["cyan"])
            barra_vida_cv(px-65, py+76, 130, 10,
                           pj["vida"], pj["vida_max"], C["verde"])
            canvas.create_text(px, py+92,
                                text=f"{pj['vida']}/{pj['vida_max']}",
                                font=F_TINY, fill=C["blanco"])
            canvas.create_text(px, py+106,
                                text=f"ATK:{pj['atk']} DEF:{pj['def']}",
                                font=F_TINY, fill=C["gris"])

        # Mini equipo jugador
        def mini_jug(idx):
            if idx >= len(eq_jug):
                return
            p   = eq_jug[idx]
            pmx = 20 + idx * 46
            pmy = 250
            av  = imgs_bat.get((p["nombre"], False))
            if av:
                canvas.create_image(pmx, pmy, anchor="nw", image=av)
            if idx == bat["idx_jug"]:
                canvas.create_rectangle(pmx, pmy, pmx+42, pmy+42,
                                         fill="", outline=C["amarillo"], width=2)
            elif p["vida"] <= 0:
                canvas.create_rectangle(pmx, pmy, pmx+42, pmy+42,
                                         fill="#000000", stipple="gray75")
                canvas.create_text(pmx+21, pmy+21, text="KO",
                                    font=F_TINY, fill=C["rojo"])
            mini_jug(idx + 1)

        mini_jug(0)

        # Panel hollow
        canvas.create_text(ANCHO-20, 44,
                            text=f"HOLLOW  PTS:{bat['pts_hol']}",
                            font=F_TINY, fill=C["rojo"], anchor="e")
        if ph:
            px, py = ANCHO-170, 128
            img_ph = imgs_bat.get((ph["nombre"], True))
            if img_ph:
                canvas.create_image(px, py, anchor="center", image=img_ph)
            canvas.create_text(px, py+62, text=ph["nombre"],
                                font=F_SMALL, fill=C["rojo"])
            barra_vida_cv(px-65, py+76, 130, 10,
                           ph["vida"], ph["vida_max"], C["rojo"])
            canvas.create_text(px, py+92,
                                text=f"{ph['vida']}/{ph['vida_max']}",
                                font=F_TINY, fill=C["blanco"])
            canvas.create_text(px, py+106,
                                text=f"ATK:{ph['atk']} DEF:{ph['def']}",
                                font=F_TINY, fill=C["gris"])

        # Mini equipo hollow
        def mini_hol(idx):
            if idx >= len(eq_hol):
                return
            p   = eq_hol[idx]
            pmx = ANCHO - 20 - (idx+1)*46
            pmy = 250
            av  = imgs_bat.get((p["nombre"], True))
            if av:
                canvas.create_image(pmx, pmy, anchor="nw", image=av)
            if idx == bat["idx_hol"]:
                canvas.create_rectangle(pmx, pmy, pmx+42, pmy+42,
                                         fill="", outline=C["rojo"], width=2)
            elif p["vida"] <= 0:
                canvas.create_rectangle(pmx, pmy, pmx+42, pmy+42,
                                         fill="#000000", stipple="gray75")
                canvas.create_text(pmx+21, pmy+21, text="KO",
                                    font=F_TINY, fill=C["gris"])
            mini_hol(idx + 1)

        mini_hol(0)

        texto_sombra(ANCHO//2, 138, "VS", F_MED, C["amarillo"])

        # Log
        lx, ly = ANCHO//2-200, 298
        canvas.create_rectangle(lx, ly, lx+400, ly+104,
                                  fill=C["panel"], outline="#3C1E50")

        def dibujar_log(msgs, idx):
            if idx >= len(msgs):
                return
            col = C["amarillo"] if idx == len(msgs)-1 else C["gris"]
            canvas.create_text(lx+8, ly+10+idx*20,
                                text=msgs[idx],
                                font=F_TINY, fill=col, anchor="w")
            dibujar_log(msgs, idx + 1)

        dibujar_log(bat["log"][-5:], 0)

        # Botones según modo y estado
        if bat["fin"] is None:
            if bat["modo"] == "accion" and bat["turno"] == "jugador":
                dibujar_botones_accion()
            elif bat["turno"] == "jugador" and bat["modo"] == "cambio":
                dibujar_panel_cambio()

        # Fin de batalla
        if bat["fin"]:
            canvas.create_rectangle(0, 0, ANCHO, ALTO,
                                     fill="#000000", stipple="gray50")
            if bat["fin"] == "jugador":
                texto_sombra(ANCHO//2, ALTO//2-60,
                             "VICTORIA!", F_TITULO, C["amarillo"])
                canvas.create_text(
                    ANCHO//2, ALTO//2+10,
                    text=f"Hollow derrotado en {UBICACIONES[ubi_idx]}",
                    font=F_MED, fill=C["cyan"])
            else:
                texto_sombra(ANCHO//2, ALTO//2-60,
                             "DERROTA", F_TITULO, C["rojo"])
                canvas.create_text(
                    ANCHO//2, ALTO//2+10,
                    text="El Hollow te vencio...",
                    font=F_MED, fill=C["gris"])
            boton_cv(ANCHO//2, ALTO//2+82, 250, 46, "CONTINUAR >",
                      C["magenta"], C["magenta"], F_MED,
                      ir_resultado, "btn_cont")

    
    #  BOTONES DE ACCIÓN  (4 botones: ATACAR | DEFENDER | CAMBIAR | RENDIRSE)
    
    def dibujar_botones_accion():
        # Espaciado para 4 botones
        boton_cv(ANCHO//2 - 240, 454, 130, 38, "ATACAR",
                  C["magenta"], C["magenta"], F_SMALL,
                  accion_atacar, "btn_atk")
        boton_cv(ANCHO//2 - 80, 454, 130, 38, "DEFENDER",
                  C["azul"], C["azul"], F_SMALL,
                  accion_defender, "btn_def")
        boton_cv(ANCHO//2 + 80, 454, 130, 38, "CAMBIAR",
                  C["cyan"], C["cyan"], F_SMALL,
                  abrir_cambio, "btn_cam")
        boton_cv(ANCHO//2 + 240, 454, 130, 38, "RENDIRSE",
                  C["rojo"], C["rojo"], F_SMALL,
                  accion_rendirse, "btn_rend")


    #  PANEL DE CAMBIO DE PERSONAJE
    
    def dibujar_panel_cambio():
        canvas.create_rectangle(ANCHO//2-280, 405, ANCHO//2+280, 620,
                                  fill="#08031A", outline=C["cyan"], width=2)
        canvas.create_text(ANCHO//2, 418,
                            text="SELECCIONA TU PERSONAJE:",
                            font=F_TINY, fill=C["cyan"])

        disponibles = []
        def buscar_disp(idx):
            if idx >= len(eq_jug):
                return
            if eq_jug[idx]["vida"] > 0 and idx != bat["idx_jug"]:
                disponibles.append(idx)
            buscar_disp(idx + 1)
        buscar_disp(0)

        if not disponibles:
            canvas.create_text(ANCHO//2, 490,
                                text="No tienes mas personajes disponibles",
                                font=F_TINY, fill=C["gris"])
        else:
            btn_w  = 120
            total  = len(disponibles) * btn_w + (len(disponibles)-1) * 10
            start  = ANCHO//2 - total//2

            def dibujar_opcion(disp_idx):
                if disp_idx >= len(disponibles):
                    return
                idx_p = disponibles[disp_idx]
                p     = eq_jug[idx_p]
                bx    = start + disp_idx * (btn_w + 10)
                by    = 432
                bh    = 130

                # ── CORRECCIÓN: usar p["avatar"] directamente ──
                mini  = cargar_img(p["avatar"], (60, 60))
                tag   = f"opt_{disp_idx}"

                r = canvas.create_rectangle(bx, by, bx+btn_w, by+bh,
                                             fill=C["card_bg"],
                                             outline=C["cyan"],
                                             width=1, tags=tag)
                ii = canvas.create_image(bx+btn_w//2, by+36,
                                          anchor="center",
                                          image=mini, tags=tag)
                nm = canvas.create_text(bx+btn_w//2, by+74,
                                         text=p["nombre"][:10],
                                         font=F_TINY, fill=C["blanco"],
                                         tags=tag)
                hp = canvas.create_text(bx+btn_w//2, by+90,
                                         text=f"HP:{p['vida']}/{p['vida_max']}",
                                         font=F_TINY, fill=C["verde"],
                                         tags=tag)
                st = canvas.create_text(bx+btn_w//2, by+106,
                                         text=f"A:{p['atk']} D:{p['def']}",
                                         font=F_TINY, fill=C["gris"],
                                         tags=tag)

                canvas.tag_bind(tag, "<Enter>",
                    lambda e, rr=r: canvas.itemconfig(rr,
                                    outline=C["amarillo"], width=2))
                canvas.tag_bind(tag, "<Leave>",
                    lambda e, rr=r: canvas.itemconfig(rr,
                                    outline=C["cyan"], width=1))
                canvas.tag_bind(tag, "<Button-1>",
                    lambda e, i=idx_p: ejecutar_cambio(i))

                dibujar_opcion(disp_idx + 1)

            dibujar_opcion(0)

        boton_cv(ANCHO//2, 588, 160, 28, "CANCELAR",
                  C["gris"], C["gris"], F_TINY,
                  cerrar_cambio, "btn_cancel")

    def abrir_cambio():
        bat["modo"] = "cambio"
        render()

    def cerrar_cambio():
        bat["modo"] = "accion"
        render()

    def ejecutar_cambio(nuevo_idx):
        anterior = eq_jug[bat["idx_jug"]]["nombre"]
        bat["idx_jug"] = nuevo_idx
        nuevo    = eq_jug[bat["idx_jug"]]["nombre"]
        agregar_log(f"Cambio: {anterior} -> {nuevo}!")
        bat["modo"]       = "accion"
        bat["turno"]      = "hollow"
        bat["defendiendo"] = False
        render()
        aid = root.after(950, turno_hollow)
        _after_ids.append(aid)

    # ══════════════════════════════════════════════════════
    #  ACCIONES DE BATALLA
    # ══════════════════════════════════════════════════════
    def accion_atacar():
        if bat["fin"] or bat["turno"] != "jugador":
            return
        pj = eq_jug[bat["idx_jug"]]
        ph = eq_hol[bat["idx_hol"]]
        d  = calcular_dano(pj["atk"], ph["def"])
        ph["vida"] = max(0, ph["vida"] - d)
        agregar_log(f"{pj['nombre']} ataca! -{d} HP a {ph['nombre']}")

        if ph["vida"] <= 0:
            capturado = dict(ph)
            capturado["vida"] = capturado["vida_max"]
            eq_jug.append(capturado)
            # Precargar imagen del capturado
            ruta = capturado["avatar"]
            imgs_bat[(capturado["nombre"], False)] = cargar_img(ruta, (100,100), False)
            imgs_bat[(capturado["nombre"], True)]  = cargar_img(ruta, (100,100), True)
            agregar_log(f"Capturaste a {ph['nombre']}! (vida restaurada)")
            bat["pts_jug"] += 1

            if not hay_vivos_hol():
                bat["fin"] = "jugador"
                render()
                return
            else:
                sig = primer_vivo_hol(excluir=bat["idx_hol"])
                if sig != -1:
                    bat["idx_hol"] = sig

        bat["turno"]       = "hollow"
        bat["modo"]        = "accion"
        bat["defendiendo"] = False
        render()
        if not bat["fin"]:
            aid = root.after(950, turno_hollow)
            _after_ids.append(aid)

    def accion_defender():
        """
        El jugador elige DEFENDER: activa el escudo para este turno.
        El daño recibido del Hollow se reduce a la mitad (min 1).
        Cuenta como acción: pasa el turno al Hollow.
        """
        if bat["fin"] or bat["turno"] != "jugador":
            return
        pj = eq_jug[bat["idx_jug"]]
        bat["defendiendo"] = True
        agregar_log(f"{pj['nombre']} se pone en defensa!")
        bat["turno"] = "hollow"
        bat["modo"]  = "accion"
        render()
        aid = root.after(950, turno_hollow)
        _after_ids.append(aid)

    def accion_rendirse():
        bat["fin"] = "hollow"
        render()

    def turno_hollow():
        """El hollow actua completamente al azar: ataca o cambia."""
        if bat["fin"] or bat["turno"] != "hollow":
            return

        pj = eq_jug[bat["idx_jug"]]
        ph = eq_hol[bat["idx_hol"]]

        opciones = ["atacar", "atacar", "atacar"]

        disp_hol = []
        def buscar_hol(idx):
            if idx >= len(eq_hol):
                return
            if eq_hol[idx]["vida"] > 0 and idx != bat["idx_hol"]:
                disp_hol.append(idx)
            buscar_hol(idx + 1)
        buscar_hol(0)

        if disp_hol:
            opciones.append("cambiar")

        accion = random.choice(opciones)

        if accion == "atacar":
            d_base = calcular_dano(ph["atk"], pj["def"])
            # Si el jugador está defendiendo, daño reducido a la mitad
            if bat["defendiendo"]:
                d = max(1, d_base // 2)
                agregar_log(f"HOLLOW ataca! Bloqueado! -{d} HP a {pj['nombre']}")
            else:
                d = d_base
                agregar_log(f"HOLLOW ataca! -{d} HP a {pj['nombre']}")

            pj["vida"] = max(0, pj["vida"] - d)

            if pj["vida"] <= 0:
                agregar_log(f"{pj['nombre']} quedo en KO!")
                bat["pts_hol"] += 1

                if not hay_vivos_jug():
                    bat["defendiendo"] = False
                    bat["fin"] = "hollow"
                    render()
                    return
                else:
                    sig = primer_vivo_jug(excluir=bat["idx_jug"])
                    if sig != -1:
                        bat["idx_jug"] = sig
                        agregar_log(f"Entra {eq_jug[bat['idx_jug']]['nombre']}!")

        elif accion == "cambiar" and disp_hol:
            nuevo = random.choice(disp_hol)
            agregar_log(f"HOLLOW cambia a {eq_hol[nuevo]['nombre']}!")
            bat["idx_hol"] = nuevo

        # Siempre resetear la defensa al final del turno del hollow
        bat["defendiendo"] = False
        bat["turno"] = "jugador"
        bat["modo"]  = "accion"
        render()

    def ir_resultado():
        estado["puntaje"] = bat["pts_jug"]
        if bat["fin"] == "jugador":
            estado["victorias"] += 1
            estado["ubicacion"] += 1
            def restaurar(idx):
                if idx >= len(eq_jug):
                    return
                eq_jug[idx]["vida"] = eq_jug[idx]["vida_max"]
                restaurar(idx + 1)
            restaurar(0)
            estado["equipo"] = eq_jug
            if estado["ubicacion"] >= 5:
                pantalla_fin()
            else:
                pantalla_mapa()
        else:
            pantalla_fin()

    render()

#  FIN

def pantalla_fin():
    limpiar()
    dibujar_fondo(FB_NOMBRE)

    ent       = ENTRENADORES[estado["entrenador"]]
    victorias = estado["victorias"]
    puntaje   = estado["puntaje"]
    nombre    = estado["nombre"]

    img_ent_fin = cargar_img(buscar_img(ent["avatar"]), (130, 162))
    canvas.create_image(ANCHO//2-280, ALTO//2-20,
                         anchor="center", image=img_ent_fin)

    cx = ANCHO//2 + 70

    if victorias >= 5:
        texto_sombra(cx, 148, "FELICITACIONES!", F_TITULO, C["amarillo"])
        canvas.create_text(cx, 220,
                            text="Derrotaste al Hollow en todas",
                            font=F_MED, fill=C["cyan"])
        canvas.create_text(cx, 246,
                            text="las ubicaciones de Cyber City!",
                            font=F_MED, fill=C["cyan"])
    else:
        texto_sombra(cx, 148, "GAME OVER", F_TITULO, C["rojo"])
        canvas.create_text(cx, 228,
                            text="El Hollow domina Cyber City...",
                            font=F_MED, fill=C["gris"])

    canvas.create_text(cx, 300,
                        text=f"JUGADOR: {nombre}",
                        font=F_MED, fill=C["blanco"])
    canvas.create_text(cx, 332,
                        text=f"ENTRENADOR: {ent['nombre']}",
                        font=F_SMALL, fill=ent["color"])
    canvas.create_text(cx, 364,
                        text=f"PUNTAJE FINAL: {puntaje}",
                        font=F_MED, fill=C["amarillo"])
    canvas.create_text(cx, 396,
                        text=f"UBICACIONES: {victorias}/5",
                        font=F_MED, fill=C["cyan"])

    def reiniciar():
        estado.update({
            "nombre": "", "entrenador": 0, "equipo": [],
            "ubicacion": 0, "victorias": 0, "puntaje": 0
        })
        pantalla_nombre()

    boton_cv(cx-130, 472, 210, 44, "SALIR",
             C["rojo"], C["rojo"], F_SMALL,
             lambda: root.quit(), "btn_salir")
    boton_cv(cx+130, 472, 250, 44, "JUGAR DE NUEVO",
             C["magenta"], C["magenta"], F_SMALL,
             reiniciar, "btn_nuevo")


#  ARRANQUe

pantalla_nombre()
root.mainloop()
