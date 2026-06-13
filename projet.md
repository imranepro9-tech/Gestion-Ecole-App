
# Gestion-Ecole-Appimport pyodbc
import tkinter as tk
from tkinter import messagebox
from tkinter import ttk
import re
import os

# اسم الملف النصي اللي غيتخبا فيه السيرفر وسط الدوسي د البروجي
CONFIG_FILE = "config_server.txt"

# دالة ذكية: كتقرا السيرفر لي مخبي، ومالقاتوش كتحط السيرفر ديالك الافتراضي
def get_server_config():
    if os.path.exists(CONFIG_FILE):
        try:
            with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                lines = f.read().splitlines()
                if len(lines) >= 2:
                    return lines[0].strip(), lines[1].strip()
        except:
            pass
    # السيرفر الافتراضي ديالك كيعمر بوحدو أول مرة
    return r"DESKTOP-V8UR5QV\SQLEXPRESS", "Gestion_ecole"

# دالة حفظ الإعدادات ف الملف النصي (Sauvegarder)
def sauvegarder_config(nouveau_server, nouvelle_db, fenetre_config):
    if nouveau_server.strip() == "" or nouvelle_db.strip() == "":
        messagebox.showwarning("Attention", "Veuillez remplir tous les champs !")
        return
    try:
        with open(CONFIG_FILE, "w", encoding="utf-8") as f:
            f.write(f"{nouveau_server.strip()}\n{nouvelle_db.strip()}")
        messagebox.showinfo("Succès", "⚙️ Configuration enregistrée et sauvegardée définitivement !")
        fenetre_config.destroy()
    except Exception as e:
        messagebox.showerror("Erreur", f"Impossible d'enregistrer la configuration :\n{e}")

# نافذة إعدادات السيرفر
def ouvrir_fenetre_config():
    server_actuel, db_actuelle = get_server_config()
    
    win_cfg = tk.Toplevel(window_login)
    win_cfg.title("🔧 Configuration Serveur SQL")
    win_cfg.geometry("350x220")
    win_cfg.configure(bg="#212529") # Dark background
    win_cfg.grab_set() 
    
    tk.Label(win_cfg, text="Nom du Serveur SQL :", font=("Arial", 10, "bold"), bg="#212529", fg="#f8f9fa").pack(pady=5)
    ent_srv = tk.Entry(win_cfg, width=40, bg="#343a40", fg="white", insertbackground="white", relief="solid", bd=1)
    ent_srv.pack(pady=2)
    ent_srv.insert(0, server_actuel)
    
    tk.Label(win_cfg, text="Nom de la Base de Données :", font=("Arial", 10, "bold"), bg="#212529", fg="#f8f9fa").pack(pady=5)
    ent_db = tk.Entry(win_cfg, width=40, bg="#343a40", fg="white", insertbackground="white", relief="solid", bd=1)
    ent_db.pack(pady=2)
    ent_db.insert(0, db_actuelle)
    
    tk.Button(win_cfg, text="💾 Sauvegarder Config", bg="#0d6efd", fg="white", font=("Arial", 10, "bold"), relief="flat", activebackground="#0b5ed7", activeforeground="white").pack(pady=15)

# 1. الاتصال الديناميكي بقاعدة البيانات
def connect_db():
    server, database = get_server_config()
    return pyodbc.connect(
        f'DRIVER={{SQL Server}};'
        f'SERVER={server};'
        f'DATABASE={database};'
        f'Trusted_Connection=yes;'
        f'TrustServerCertificate=yes;'
    )

# دالة لتصحيح فورمات التاريخ (JJ-MM-AAAA -> AAAA-MM-JJ)
def corriger_date(date_str):
    date_clean = date_str.strip().replace('/', '-')
    if re.match(r'^\d{2}-\d{2}-\d{4}$', date_clean):
        j, m, a = date_clean.split('-')
        return f"{a}-{m}-{j}"
    return date_clean

# 2. Vérification du Login
def verifier_login():
    user = entry_user.get().strip()
    pwd = entry_pwd.get().strip()
    
    if user == "admin" and pwd == "admin123":
        messagebox.showinfo("Succès", "🎉 Connexion réussie !")
        window_login.destroy()
        ouvrir_application()
    else:
        messagebox.showerror("Erreur", "Nom d'utilisateur ou mot de passe incorrect !")

# 3. Fonction d'insertion (INSERT)
def enregistrer_table(nom_table, fields, entries):
    values = [entry.get() for entry in entries]
    if any(v == "" for v in values):
        messagebox.showwarning("Attention", "Veuillez remplir tous les champs !")
        return
        
    if nom_table == "Etudiant":
        try:
            idx_date = fields.index("date_naissance")
            values[idx_date] = corriger_date(values[idx_date])
        except ValueError:
            pass
        
    try:
        conn = connect_db()
        cursor = conn.cursor()
        placeholders = ", ".join(["?"] * len(values))
        columns = ", ".join(fields)
        query = f"INSERT INTO {nom_table} ({columns}) VALUES ({placeholders})"
        
        cursor.execute(query, values)
        conn.commit()
        messagebox.showinfo("Succès", f"✅ Données enregistrées dans {nom_table} !")
        for entry in entries: entry.delete(0, tk.END)
    except Exception as e:
        messagebox.showerror("Erreur SQL", f"Erreur lors de l'insertion :\n{e}")
    finally:
        if 'conn' in locals(): conn.close()

# 4. Fonction de modification (UPDATE)
def modifier_table(nom_table, fields, entries):
    values = [entry.get() for entry in entries]
    if any(v == "" for v in values):
        messagebox.showwarning("Attention", "Veuillez remplir tous les champs !")
        return

    if nom_table == "Etudiant":
        try:
            idx_date = fields.index("date_naissance")
            values[idx_date] = corriger_date(values[idx_date])
        except ValueError:
            pass

    try:
        conn = connect_db()
        cursor = conn.cursor()
        
        pk_field = fields[0]
        pk_value = values[0]
        
        update_pairs = [f"{field} = ?" for field in fields[1:]]
        update_clause = ", ".join(update_pairs)
        
        query = f"UPDATE {nom_table} SET {update_clause} WHERE {pk_field} = ?"
        update_values = values[1:] + [pk_value]
        
        cursor.execute(query, update_values)
        conn.commit()
        
        if cursor.rowcount > 0:
            messagebox.showinfo("Succès", f"🔄 Données modifiées dans {nom_table} !")
            for entry in entries: entry.delete(0, tk.END)
        else:
            messagebox.showwarning("Attention", f"ID non trouvé : {pk_value}")
    except Exception as e:
        messagebox.showerror("Erreur", f"Modification impossible :\n{e}")
    finally:
        if 'conn' in locals(): conn.close()

# 5. Fonction de suppression (DELETE)
def supprimer_row(nom_table, pk_field, entry_id):
    pk_value = entry_id.get()
    if pk_value == "":
        messagebox.showwarning("Attention", "Entrez l'ID à supprimer !")
        return
        
    if not messagebox.askyesno("Confirmation", f"Voulez-vous supprimer cette ligne de {nom_table} ?"):
        return

    try:
        conn = connect_db()
        cursor = conn.cursor()
        query = f"DELETE FROM {nom_table} WHERE {pk_field} = ?"
        cursor.execute(query, [pk_value])
        conn.commit()
        
        if cursor.rowcount > 0:
            messagebox.showinfo("Succès", "🗑️ Suppression réussie !")
            entry_id.delete(0, tk.END)
        else:
            messagebox.showwarning("Attention", "ID non trouvé !")
    except Exception as e:
        messagebox.showerror("Erreur", f"Suppression impossible :\n{e}")
    finally:
        if 'conn' in locals(): conn.close()

# 6. Fonction d'affichage et recherche
def charger_donnees_affichage(nom_table, tree, search_text=""):
    for item in tree.get_children():
        tree.delete(item)
    if not nom_table: return
    try:
        conn = connect_db()
        cursor = conn.cursor()
        if nom_table == "Etudiant": pk, fields = "id_etudiant", ["id_etudiant", "nom", "prenom", "date_naissance"]
        elif nom_table == "Matiere": pk, fields = "id_matiere", ["id_matiere", "nom_matiere", "coefficient"]
        elif nom_table == "Note": pk, fields = "id_note", ["id_note", "id_etudiant", "id_matiere", "note"]
        elif nom_table == "Formateur": pk, fields = "id_Formateur", ["id_Formateur", "nom", "id_matiere"]
            
        tree["columns"] = fields
        tree["show"] = "headings"
        for f in fields:
            tree.heading(f, text=f.upper())
            tree.column(f, width=110, anchor="center")
            
        if search_text:
            query = f"SELECT * FROM {nom_table} WHERE CAST({pk} AS VARCHAR) LIKE ? OR {fields[1]} LIKE ?"
            cursor.execute(query, (f"%{search_text}%", f"%{search_text}%"))
        else:
            query = f"SELECT * FROM {nom_table}"
            cursor.execute(query)
            
        for row in cursor.fetchall():
            tree.insert("", tk.END, values=[str(val) for val in row])
    except Exception as e:
        messagebox.showerror("Erreur", f"Affichage impossible :\n{e}")
    finally:
        if 'conn' in locals(): conn.close()

# 7. Fonction de calcul des moyennes et classement
def calculer_et_classer(tree_classement):
    for item in tree_classement.get_children():
        tree_classement.delete(item)
    try:
        conn = connect_db()
        cursor = conn.cursor()
        query = """
            SELECT E.id_etudiant, E.nom, E.prenom, ROUND(SUM(N.note * M.coefficient) / SUM(M.coefficient), 2) AS Moyenne
            FROM Note N
            INNER JOIN Etudiant E ON N.id_etudiant = E.id_etudiant
            INNER JOIN Matiere M ON N.id_matiere = M.id_matiere
            GROUP BY E.id_etudiant, E.nom, E.prenom
            ORDER BY Moyenne DESC
        """
        cursor.execute(query)
        position = 1
        for row in cursor.fetchall():
            row_values = [str(position)] + [str(val) for val in row]
            tree_classement.insert("", tk.END, values=row_values)
            position += 1
        if position == 1:
            messagebox.showwarning("Attention", "Aucune note trouvée !")
    except Exception as e:
        messagebox.showerror("Erreur", f"Calcul impossible :\n{e}")
    finally:
        if 'conn' in locals(): conn.close()

# 8. Fenêtre principale de l'application
def ouvrir_application():
    app = tk.Tk()
    app.title("Système de Gestion Scolaire 🏫")
    app.geometry("720x560")
    app.configure(bg="#1e1e24") # Dark Theme Background
    
    # إعداد ستايل مودرن دايرك (Styles ttk)
    style = ttk.Style()
    style.theme_use("clam")
    
    # ستايل التبويبات الفوقانية (Dark Tabs)
    style.configure("TNotebook", background="#1e1e24", borderwidth=0)
    style.configure("TNotebook.Tab", font=("Arial", 10, "bold"), padding=[12, 5], background="#2d2d34", foreground="#b0b0b5")
    style.map("TNotebook.Tab", background=[("selected", "#0d6efd")], foreground=[("selected", "white")])
    
    # ستايل الجداول (Treeview Dark)
    style.configure("Treeview", font=("Arial", 10), rowheight=26, background="#2d2d34", fieldbackground="#2d2d34", foreground="white")
    style.configure("Treeview.Heading", font=("Arial", 10, "bold"), background="#1e1e24", foreground="white", relief="flat")
    style.map("Treeview.Heading", background=[("active", "#0d6efd")])

    notebook = ttk.Notebook(app)
    notebook.pack(pady=10, fill='both', expand=True, padx=10)
    
    # TAB 1 : INSERTION
    tab_insert = ttk.Frame(notebook)
    notebook.add(tab_insert, text="➕ Insertion")
    tab_insert.configure(style="TNotebook") # تطبيق الخلفية الداكنة
    
    tk.Label(tab_insert, text="Sélectionnez la table pour insérer un enregistrement :", font=("Arial", 11, "bold"), bg="#1e1e24", fg="#f8f9fa").pack(pady=10)
    combo_insert = ttk.Combobox(tab_insert, values=["Etudiant", "Matiere", "Note", "Formateur"], state="readonly")
    combo_insert.pack(pady=5)
    
    frame_ins_inputs = tk.Frame(tab_insert, bg="#1e1e24")
    frame_ins_inputs.pack(pady=15, fill='x', padx=30)
    
    def charger_champs_insert(event):
        for widget in frame_ins_inputs.winfo_children(): widget.destroy()
        table = combo_insert.get()
        if table == "Etudiant": fields = ["id_etudiant", "nom", "prenom", "date_naissance"]
        elif table == "Matiere": fields = ["id_matiere", "nom_matiere", "coefficient"]
        elif table == "Note": fields = ["id_note", "id_etudiant", "id_matiere", "note"]
        elif table == "Formateur": fields = ["id_Formateur", "nom", "id_matiere"]
        entries = []
        for f in fields:
            row = tk.Frame(frame_ins_inputs, bg="#1e1e24")
            row.pack(pady=5, fill='x')
            tk.Label(row, text=f"{f} : ", width=18, anchor='w', font=("Arial", 10), bg="#1e1e24", fg="#ced4da").pack(side='left')
            ent = tk.Entry(row, bg="#2d2d34", fg="white", insertbackground="white", relief="solid", bd=1)
            ent.pack(side='right', expand=True, fill='x', padx=5)
            entries.append(ent)
        tk.Button(frame_ins_inputs, text=f"Enregistrer dans {table}", bg="#198754", fg="white", font=("Arial", 10, "bold"), relief="flat", activebackground="#157347", activeforeground="white").pack(pady=15)
    combo_insert.bind("<<ComboboxSelected>>", charger_champs_insert)
    
    # TAB 2 : MODIFICATION
    tab_update = ttk.Frame(notebook)
    notebook.add(tab_update, text="🔄 Modification")
    tab_update.configure(style="TNotebook")
    
    tk.Label(tab_update, text="Sélectionnez la table à modifier :", font=("Arial", 11, "bold"), bg="#1e1e24", fg="#f8f9fa").pack(pady=10)
    combo_update = ttk.Combobox(tab_update, values=["Etudiant", "Matiere", "Note", "Formateur"], state="readonly")
    combo_update.pack(pady=5)
    
    frame_up_inputs = tk.Frame(tab_update, bg="#1e1e24")
    frame_up_inputs.pack(pady=15, fill='x', padx=30)
    
    def charger_champs_update(event):
        for widget in frame_up_inputs.winfo_children(): widget.destroy()
        table = combo_update.get()
        if table == "Etudiant": fields = ["id_etudiant", "nom", "prenom", "date_naissance"]
        elif table == "Matiere": fields = ["id_matiere", "nom_matiere", "coefficient"]
        elif table == "Note": fields = ["id_note", "id_etudiant", "id_matiere", "note"]
        elif table == "Formateur": fields = ["id_Formateur", "nom", "id_matiere"]
        entries = []
        for f in fields:
            row = tk.Frame(frame_up_inputs, bg="#1e1e24")
            row.pack(pady=5, fill='x')
            label_text = f"{f} (Clé Primaire) : " if f == fields[0] else f"{f} : "
            lbl_fg = "#ffc107" if f == fields[0] else "#ced4da"
            tk.Label(row, text=label_text, width=22, anchor='w', font=("Arial", 10), bg="#1e1e24", fg=lbl_fg).pack(side='left')
            ent = tk.Entry(row, bg="#2d2d34", fg="white", insertbackground="white", relief="solid", bd=1)
            ent.pack(side='right', expand=True, fill='x', padx=5)
            entries.append(ent)
        tk.Button(frame_up_inputs, text=f"Mettre à jour dans {table}", bg="#ffc107", fg="black", font=("Arial", 10, "bold"), relief="flat", activebackground="#e0a800").pack(pady=15)
    combo_update.bind("<<ComboboxSelected>>", charger_champs_update)

    # TAB 3 : SUPPRESSION
    tab_delete = ttk.Frame(notebook)
    notebook.add(tab_delete, text="🗑️ Suppression")
    tab_delete.configure(style="TNotebook")
    
    tk.Label(tab_delete, text="Sélectionnez la table pour la suppression :", font=("Arial", 11, "bold"), bg="#1e1e24", fg="#dc3545").pack(pady=10)
    combo_delete = ttk.Combobox(tab_delete, values=["Etudiant", "Matiere", "Note", "Formateur"], state="readonly")
    combo_delete.pack(pady=5)
    
    frame_del_inputs = tk.Frame(tab_delete, bg="#1e1e24")
    frame_del_inputs.pack(pady=20)
    
    def charger_champs_delete(event):
        for widget in frame_del_inputs.winfo_children(): widget.destroy()
        table = combo_delete.get()
        if table == "Etudiant": pk_field = "id_etudiant"
        elif table == "Matiere": pk_field = "id_matiere"
        elif table == "Note": pk_field = "id_note"
        elif table == "Formateur": pk_field = "id_Formateur"
        row = tk.Frame(frame_del_inputs, bg="#1e1e24")
        row.pack(pady=10, fill='x')
        tk.Label(row, text=f"Entrez {pk_field} à supprimer : ", font=("Arial", 10), bg="#1e1e24", fg="#ced4da").pack(side='left', padx=5)
        ent_id = tk.Entry(row, width=15, bg="#2d2d34", fg="white", insertbackground="white", relief="solid", bd=1)
        ent_id.pack(side='left', padx=5)
        tk.Button(frame_del_inputs, text=f"Supprimer de {table}", bg="#dc3545", fg="white", font=("Arial", 10, "bold"), relief="flat", activebackground="#bb2d3b", activeforeground="white").pack(pady=20)
    combo_delete.bind("<<ComboboxSelected>>", charger_champs_delete)

    # TAB 4 : AFFICHAGE & RECHERCHE
    tab_view = ttk.Frame(notebook)
    notebook.add(tab_view, text="🔍 Affichage & Recherche")
    tab_view.configure(style="TNotebook")
    
    frame_top_view = tk.Frame(tab_view, bg="#1e1e24")
    frame_top_view.pack(pady=10, fill='x', padx=15)
    tk.Label(frame_top_view, text="Table :", font=("Arial", 10, "bold"), bg="#1e1e24", fg="#f8f9fa").pack(side='left', padx=5)
    combo_view = ttk.Combobox(frame_top_view, values=["Etudiant", "Matiere", "Note", "Formateur"], state="readonly", width=15)
    combo_view.pack(side='left', padx=5)
    
    tk.Label(frame_top_view, text="🔍 Recherche rapide :", bg="#1e1e24", fg="#ced4da").pack(side='left', padx=10)
    entry_search = tk.Entry(frame_top_view, width=18, bg="#2d2d34", fg="white", insertbackground="white", relief="solid", bd=1)
    entry_search.pack(side='left', padx=5)
    
    tree_frame = tk.Frame(tab_view, bg="#1e1e24")
    tree_frame.pack(pady=10, fill='both', expand=True, padx=15)
    tree = ttk.Treeview(tree_frame)
    tree.pack(side='left', fill='both', expand=True)
    scrollbar = ttk.Scrollbar(tree_frame, orient="vertical", command=tree.yview)
    scrollbar.pack(side='right', fill='y')
    tree.configure(yscrollcommand=scrollbar.set)
    def on_view_change(event): charger_donnees_affichage(combo_view.get(), tree, entry_search.get())
    combo_view.bind("<<ComboboxSelected>>", on_view_change)
    entry_search.bind("<KeyRelease>", on_view_change)

    # TAB 5 : MOYENNES & CLASSEMENT
    tab_rank = ttk.Frame(notebook)
    notebook.add(tab_rank, text="📊 Moyennes & Classement")
    tab_rank.configure(style="TNotebook")
    
    btn_calc = tk.Button(tab_rank, text="🔄 Calculer les moyennes et classer les étudiants", bg="#0d6efd", fg="white", font=("Arial", 10, "bold"), relief="flat", activebackground="#0b5ed7", activeforeground="white")
    btn_calc.configure(command=lambda: calculer_et_classer(tree_rank))
    btn_calc.pack(pady=15)
    
    rank_frame = tk.Frame(tab_rank, bg="#1e1e24")
    rank_frame.pack(pady=5, fill='both', expand=True, padx=15)
    tree_rank = ttk.Treeview(rank_frame)
    tree_rank.pack(side='left', fill='both', expand=True)
    headers_rank = ["Rang", "ID Étudiant", "Nom", "Prénom", "Moyenne Générale"]
    tree_rank["columns"] = headers_rank
    tree_rank["show"] = "headings"
    for h in headers_rank:
        tree_rank.heading(h, text=h.upper())
        tree_rank.column(h, width=105, anchor="center")
    scrollbar_rank = ttk.Scrollbar(rank_frame, orient="vertical", command=tree_rank.yview)
    scrollbar_rank.pack(side='right', fill='y')
    tree_rank.configure(yscrollcommand=scrollbar_rank.set)

    app.mainloop()

# 9. Fenêtre de Login (Dark Theme)
window_login = tk.Tk()
window_login.title("🔐 Connexion")
window_login.geometry("320x240")
window_login.configure(bg="#212529") # Dark pure background

tk.Label(window_login, text="👤 Username :", bg="#212529", fg="#ced4da", font=("Arial", 10, "bold")).pack(pady=5)
entry_user = tk.Entry(window_login, width=25, bg="#343a40", fg="white", insertbackground="white", relief="solid", bd=1, justify="center")
entry_user.pack()

tk.Label(window_login, text="🔑 Password :", bg="#212529", fg="#ced4da", font=("Arial", 10, "bold")).pack(pady=5)
entry_pwd = tk.Entry(window_login, show="*", width=25, bg="#343a40", fg="white", insertbackground="white", relief="solid", bd=1, justify="center")
entry_pwd.pack()

# بوطون تسجيل الدخول المودرن
tk.Button(window_login, text="Se connecter", width=16, bg="#198754", fg="white", font=("Arial", 10, "bold"), relief="flat", activebackground="#157347", activeforeground="white", command=verifier_login).pack(pady=10)

# بوطون الإعدادات المودرن
tk.Button(window_login, text="🔧 Config Server", width=16, bg="#6c757d", fg="white", font=("Arial", 9, "italic"), relief="flat", activebackground="#565e64", activeforeground="white", command=ouvrir_fenetre_config).pack(pady=5)

window_login.mainloop()
