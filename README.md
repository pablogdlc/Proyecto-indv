import math
import copy
import networkx as nx
import matplotlib.pyplot as plt
from matplotlib.widgets import Button
import tkinter as tk
from tkinter import ttk, messagebox

class SimuladorInteractivo:
    def __init__(self):
        # 1. ESTRUCTURA: Red de 10 personas
        self.personas = ["Toni", "Guille", "Manuel", "David", "Felipe",
                         "Fernando", "Hugo", "Héctor", "Jaime", "Juan"]
        self.n = len(self.personas)
        
        self.matriz_inicial = [[math.inf] * self.n for _ in range(self.n)]
        self.siguiente_nodo = [[None] * self.n for _ in range(self.n)]
        
        for i in range(self.n): 
            self.matriz_inicial[i][i] = 0

        self.inmunizados = []
        self.tiempo_actual = 0
        self.tiempos_llegada = {}
        self.matriz_caminos = []
        self.origen_actual = ""
        
        # Variables para controlar el final y la reproducción
        self.tiempo_maximo = 0
        self.simulacion_terminada = False
        self.reproduciendo = False  
        
        self._cargar_red_social()

    def _cargar_red_social(self):
        conexiones = [
            ("Toni", "Héctor", 40), ("Héctor", "Fernando", 40), 
            ("Fernando", "Jaime", 40), ("Jaime", "Felipe", 40), 
            ("Felipe", "Toni", 40), ("Guille", "Manuel", 20), 
            ("Manuel", "David", 20), ("David", "Juan", 20), 
            ("Juan", "Hugo", 20), ("Hugo", "Guille", 20),
            ("Toni", "Guille", 30), ("Héctor", "Manuel", 30),
            ("Fernando", "David", 30), ("Jaime", "Juan", 30),
            ("Felipe", "Hugo", 30)
        ]
        
        for p1, p2, peso in conexiones:
            i, j = self.personas.index(p1), self.personas.index(p2)
            self.matriz_inicial[i][j] = self.matriz_inicial[j][i] = peso
            self.siguiente_nodo[i][j] = j
            self.siguiente_nodo[j][i] = i

    def aplicar_floyd_warshall(self):
        dist = copy.deepcopy(self.matriz_inicial)
        caminos = copy.deepcopy(self.siguiente_nodo)
        
        for k in range(self.n):
            if self.personas[k] in self.inmunizados:
                continue 
                
            for i in range(self.n):
                for j in range(self.n):
                    if dist[i][j] > dist[i][k] + dist[k][j]:
                        dist[i][j] = dist[i][k] + dist[k][j]
                        caminos[i][j] = caminos[i][k]
        return dist, caminos

    def obtener_ruta(self, inicio_idx, fin_idx):
        if self.matriz_caminos[inicio_idx][fin_idx] is None:
            return None
        ruta = [self.personas[inicio_idx]]
        while inicio_idx != fin_idx:
            inicio_idx = self.matriz_caminos[inicio_idx][fin_idx]
            ruta.append(self.personas[inicio_idx])
        return ruta

    def iniciar_interfaz(self, origen, lista_inmunizados):
        self.origen_actual = origen
        self.inmunizados = lista_inmunizados
        self.tiempo_actual = 0
        self.simulacion_terminada = False
        self.reproduciendo = False 
        idx_origen = self.personas.index(origen)
        
        matriz_dist, self.matriz_caminos = self.aplicar_floyd_warshall()
        self.tiempos_llegada = {self.personas[i]: matriz_dist[idx_origen][i] for i in range(self.n)}
        
        tiempos_posibles = [t for t in self.tiempos_llegada.values() if t != math.inf]
        self.tiempo_maximo = max(tiempos_posibles) if tiempos_posibles else 0

        self.G = nx.Graph()
        for i in range(self.n):
            for j in range(i+1, self.n):
                if self.matriz_inicial[i][j] != math.inf:
                    self.G.add_edge(self.personas[i], self.personas[j], weight=self.matriz_inicial[i][j])

        self.fig, self.ax = plt.subplots(figsize=(12, 8))
        self.fig.canvas.manager.set_window_title('Simulador de Virus - Gráfico')
        plt.subplots_adjust(bottom=0.2) 
        
        self.pos = nx.spring_layout(self.G, seed=42)
        
        print(f"\n Origen: {self.origen_actual} | Inmune/s: {self.inmunizados}")
        self.actualizar_pantalla()
        
        ax_manual = plt.axes([0.25, 0.04, 0.22, 0.06]) 
        self.btn_manual = Button(ax_manual, 'Avanzar +10 Min', color='#2ecc71', hovercolor='#27ae60')
        self.btn_manual.on_clicked(self.clic_avanzar_tiempo)
        
        
        ax_play = plt.axes([0.53, 0.04, 0.22, 0.06])
        self.btn_play = Button(ax_play, 'Play (Auto)', color='#3498db', hovercolor='#2980b9')
        self.btn_play.on_clicked(self.clic_play)
        
        plt.show()

    def avanzar_tiempo_logica(self):
        if self.simulacion_terminada:
            return

        self.tiempo_actual += 10
        idx_origen = self.personas.index(self.origen_actual)
        
        for persona, tiempo_nodo in self.tiempos_llegada.items():
            if tiempo_nodo == self.tiempo_actual:
                idx_destino = self.personas.index(persona)
                ruta = self.obtener_ruta(idx_origen, idx_destino)
                if ruta:
                    cadena_ruta = " -> ".join(ruta)
                    estado = "(persona inmune)" if persona in self.inmunizados else ""
                    print(f"- {persona:<10} : {tiempo_nodo} min. Recorrido: {cadena_ruta} {estado}")
                    
        self.actualizar_pantalla()

        if self.tiempo_actual >= self.tiempo_maximo:
            self.simulacion_terminada = True
            self.reproduciendo = False
            self.btn_manual.color = '#bdc3c7'
            self.btn_manual.hovercolor = '#bdc3c7'
            self.btn_play.color = '#bdc3c7'
            self.btn_play.hovercolor = '#bdc3c7'
            self.btn_manual.label.set_text("Finalizado")
            self.btn_play.label.set_text("Finalizado")
            self.fig.canvas.draw_idle()

    def clic_avanzar_tiempo(self, event):
        if self.simulacion_terminada: return
        self.avanzar_tiempo_logica()

    def clic_play(self, event):
        if self.simulacion_terminada: return
        
        if self.reproduciendo:
            self.reproduciendo = False
            # Corregido: Texto estándar sin emojis conflictivos
            self.btn_play.label.set_text("Play (Auto)")
            self.fig.canvas.draw_idle()
        else:
            self.reproduciendo = True
            self.btn_play.label.set_text("Pausa")
            self.fig.canvas.draw_idle()
            
            while self.reproduciendo and not self.simulacion_terminada:
                if not plt.fignum_exists(self.fig.number): 
                    break
                self.avanzar_tiempo_logica()
                plt.pause(1.0) 

    def actualizar_pantalla(self):
        self.ax.clear() 
        colores = []
        for nodo in self.G.nodes:
            t_nodo = self.tiempos_llegada[nodo]
            if nodo in self.inmunizados: colores.append('#2c3e50') 
            elif t_nodo == 0: colores.append('#e74c3c') 
            elif self.tiempo_actual >= t_nodo and t_nodo != math.inf: colores.append("#248b08") 
            else: colores.append('#bdc3c7') 

        nx.draw_networkx_edges(self.G, self.pos, ax=self.ax, width=2.8, edge_color='#47525e', alpha=0.8)
        nx.draw_networkx_nodes(self.G, self.pos, ax=self.ax, node_color=colores, node_size=2600)
        nx.draw_networkx_labels(self.G, self.pos, ax=self.ax, font_size=10, font_weight='bold', font_color='white')
        
        labels = nx.get_edge_attributes(self.G, 'weight')
        nx.draw_networkx_edge_labels(self.G, self.pos, ax=self.ax, edge_labels=labels, font_color='#111111')
        self.ax.set_title(f"Simulador en Tiempo Real | Tiempo transcurrido: {self.tiempo_actual} min", fontsize=13, fontweight='bold')
        self.ax.axis('off')
        self.fig.canvas.draw_idle()



class AppMenuGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Simulación de propagación de un virus - Panel de Control")
        self.root.state('zoomed')
        
        self.COLOR_BG = "#F8FAFC"          
        self.COLOR_CARD = "#FFFFFF"        
        self.COLOR_TEXT = "#0F172A"        
        self.COLOR_MUTED = "#64748B"       
        self.COLOR_PRIMARY = "#4F46E5"     
        self.COLOR_ACCENT = "#10B981"      
        
        self.root.configure(bg=self.COLOR_BG)
        
        self.simulador_base = SimuladorInteractivo()
        self.personas = self.simulador_base.personas

        self.style = ttk.Style()
        self.style.theme_use('clam')
        self.style.configure("TCombobox", fieldbackground=self.COLOR_CARD, background=self.COLOR_BG)
        self.root.option_add('*TCombobox*Listbox.font', ('Verdana', 14))

        lbl_titulo = tk.Label(root, text="🦠 PANEL DE CONTROL - SIMULADOR DE VIRUS", font=("Verdana", 24, "bold"), 
                              fg=self.COLOR_TEXT, bg=self.COLOR_BG, pady=40)
        lbl_titulo.pack()

        card_frame = tk.Frame(root, bg=self.COLOR_CARD, bd=0, highlightthickness=1, highlightbackground="#E2E8F0")
        card_frame.pack(fill="both", expand=True, padx=80, pady=10)

        lbl_sec1 = tk.Label(card_frame, text="1. Seleccionar Paciente Cero", font=("Verdana", 14, "bold"), 
                            fg=self.COLOR_TEXT, bg=self.COLOR_CARD)
        lbl_sec1.pack(anchor="w", padx=40, pady=(30, 10))
        
        self.combo_origen = ttk.Combobox(card_frame, values=self.personas, state="readonly", font=("Verdana", 14))
        self.combo_origen.set("--- Elige una persona ---") 
        self.combo_origen.pack(fill="x", padx=40, pady=(0, 30))

        lbl_sec2 = tk.Label(card_frame, text="2. Personas Inmunizadas", font=("Verdana", 14, "bold"), 
                            fg=self.COLOR_TEXT, bg=self.COLOR_CARD)
        lbl_sec2.pack(anchor="w", padx=40, pady=(10, 2))
        
        lbl_ayuda = tk.Label(card_frame, text="Clica en la persona para inmunizar", 
                             font=("Verdana", 10), fg=self.COLOR_MUTED, bg=self.COLOR_CARD)
        lbl_ayuda.pack(anchor="w", padx=40, pady=(0, 10))
        
        self.lista_inmunes = tk.Listbox(
            card_frame, selectmode=tk.MULTIPLE, height=10, font=("Verdana", 14),
            bg=self.COLOR_BG, fg=self.COLOR_TEXT, bd=0, highlightthickness=1, 
            highlightbackground="#CBD5E1", selectbackground="#E0E7FF", selectforeground=self.COLOR_PRIMARY,
            exportselection=False
        )
        for p in self.personas:
            self.lista_inmunes.insert(tk.END, f"  {p}") 
        self.lista_inmunes.pack(fill="both", expand=True, padx=40, pady=(0, 40))

        btn_lanzar = tk.Button(
            root, text="EMPEZAR SIMULACIÓN", font=("Verdana", 18, "bold"), 
            bg=self.COLOR_ACCENT, fg="white", activebackground="#059669", activeforeground="white",
            bd=0, relief="flat", cursor="hand2",
            command=self.ejecutar_simulacion
        )
        btn_lanzar.pack(fill="x", padx=80, pady=40, ipady=18)

    def ejecutar_simulacion(self):
        origen_limpio = self.combo_origen.get().strip()
        
        if origen_limpio not in self.personas:
            messagebox.showwarning("Error", "Por favor, seleccione quién es el Paciente Cero antes de lanzar la simulación.")
            return
        
        indices_seleccionados = self.lista_inmunes.curselection()
        inmunizados = [self.personas[i] for i in indices_seleccionados]
        
        if origen_limpio in inmunizados:
            messagebox.showwarning("Problema", f"{origen_limpio} no puede ser el paciente cero y estar inmune al mismo tiempo.")
            return

        sim_ejecucion = SimuladorInteractivo()
        sim_ejecucion.iniciar_interfaz(origen_limpio, inmunizados)

if __name__ == "__main__":
    ventana_principal = tk.Tk()
    app = AppMenuGUI(ventana_principal)
    ventana_principal.mainloop()
