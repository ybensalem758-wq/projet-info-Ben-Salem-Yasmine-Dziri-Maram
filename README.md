import math
import matplotlib.pyplot as plt
import random
import pandas as pd  # <-- import Pandas

def generer_fichier_test(nom_fichier="data_small.csv", n_souris=20):
    """
    Génère un fichier CSV respectant la structure exacte demandée.
    Colonnes : 
    0:strain, 1:exp_id, 2:sample_type, 3:timepoint, 4:mouse_id, 
    5:treatment, 6:freq, 7:exp_day, 8:count, 9:age_days, 10:sex
    """
    print(f"Génération du fichier de test : {nom_fichier} ")
    f = open(nom_fichier, "w")
    f.write("mouse_strain;experiment_id;sample_type;timepoint;mouse_id;treatment;"
            "frequency_live_bacteria;experimental_day;counts_live_bacteria;mouse_age_days;mouse_sex\n")
    
    jours_exp = [14, 15, 16, 17, 18, 19, 20, 21, 23, 25, 28]
    
    for i in range(1, n_souris + 1):
        prefix = "M"
        if i < 10: m_id = f"{prefix}00{i}"
        elif i < 100: m_id = f"{prefix}0{i}"
        else: m_id = f"{prefix}{i}"
            
        traitement = "ABX" if i <= n_souris / 2 else "PLB"
        
        # Données fécales
        for jour_vie in jours_exp:
            jour_exp = jour_vie - 14
            base = 10**8
            valeur = base
            if traitement == "ABX":
                if 14 <= jour_vie <= 20: valeur = base / 10000
                else: valeur = base / 100
            valeur = valeur * random.uniform(0.5, 1.5)
            ligne = f"C57BL6;EXP01;fecal;TP{jour_vie};{m_id};{traitement};0.9;{jour_exp};{int(valeur)};{jour_vie};M"
            f.write(ligne + "\n")

        # Données cécales et iléales (Sacrifice J21)
        val_cecal = 10**7 if traitement == "PLB" else 10**4
        val_ileal = 10**6 if traitement == "PLB" else 10**3
        val_cecal *= random.uniform(0.2, 5.0)
        val_ileal *= random.uniform(0.2, 5.0)

        f.write(f"C57BL6;EXP01;cecal;SACRIFICE;{m_id};{traitement};0.8;7;{int(val_cecal)};21;M\n")
        f.write(f"C57BL6;EXP01;ileal;SACRIFICE;{m_id};{traitement};0.8;7;{int(val_ileal)};21;M\n")
            
    f.close()
    print("Fichier généré avec succès.\n")


def analyser_donnees(nom_fichier_entree):
    print(f"Analyse du fichier avec Pandas : {nom_fichier_entree}")
    
    # Lecture CSV
    df = pd.read_csv(nom_fichier_entree, sep=';')
    
    # Calcul Log10 du count
    df['log_count'] = df['counts_live_bacteria'].apply(lambda x: math.log10(x) if x > 0 else 0)
    
    # Fécal
    df_fecal = df[df['sample_type'] == 'fecal'][['mouse_id', 'treatment', 'experimental_day', 'counts_live_bacteria']]
    df_fecal.to_csv("filtered_fecal.csv", sep=';', index=False)
    
    # Préparation graphique fécal
    donnees_courbes = {}
    for _, row in df[df['sample_type'] == 'fecal'].iterrows():
        m_id = row['mouse_id']
        if m_id not in donnees_courbes:
            color = 'red' if row['treatment'] == 'ABX' else 'blue'
            donnees_courbes[m_id] = {'x': [], 'y': [], 'c': color}
        donnees_courbes[m_id]['x'].append(row['experimental_day'])
        donnees_courbes[m_id]['y'].append(row['log_count'])
    
    # Cécal
    df_cecal = df[(df['sample_type'] == 'cecal') & (df['mouse_age_days'] == 21)][['treatment', 'counts_live_bacteria']]
    df_cecal.to_csv("filtered_cecal.csv", sep=';', index=False)
    
    cecal_abx = df_cecal[df_cecal['treatment'] == 'ABX']['counts_live_bacteria'].apply(lambda x: math.log10(x)).tolist()
    cecal_plb = df_cecal[df_cecal['treatment'] == 'PLB']['counts_live_bacteria'].apply(lambda x: math.log10(x)).tolist()
    
    # Iléal 
    df_ileal = df[(df['sample_type'] == 'ileal') & (df['mouse_age_days'] == 21)][['treatment', 'counts_live_bacteria']]
    df_ileal.to_csv("filtered_ileal.csv", sep=';', index=False)
    
    ileal_abx = df_ileal[df_ileal['treatment'] == 'ABX']['counts_live_bacteria'].apply(lambda x: math.log10(x)).tolist()
    ileal_plb = df_ileal[df_ileal['treatment'] == 'PLB']['counts_live_bacteria'].apply(lambda x: math.log10(x)).tolist()
    
    print("Fichiers CSV filtrés générés (filtered_fecal.csv, filtered_cecal.csv, filtered_ileal.csv)")
    
    # Graphique 1 : Courbes fécal 
    plt.figure(figsize=(10, 6))
    for m_id in donnees_courbes:
        d = donnees_courbes[m_id]
        points = sorted(zip(d['x'], d['y']))
        x_sorted = [p[0] for p in points]
        y_sorted = [p[1] for p in points]
        plt.plot(x_sorted, y_sorted, color=d['c'], alpha=0.6, linewidth=1.5)

    from matplotlib.lines import Line2D
    legend_elements = [Line2D([0], [0], color='blue', lw=2, label='Placebo'),
                       Line2D([0], [0], color='red', lw=2, label='Antibiotique')]
    plt.legend(handles=legend_elements)
    plt.title("Évolution fécale (Log10)")
    plt.xlabel("Jour expérimental")
    plt.ylabel("Bactéries (Log10 CFU)")
    plt.grid(True, alpha=0.3)
    plt.savefig("graphique_lignes_fecal.png")
    print("Graphique lignes généré : graphique_lignes_fecal.png")

    # --- Graphique 2 : Violons (Cécal & Iléal) ---
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 6))

    # Cécal 
    data_cecal = [cecal_abx, cecal_plb]
    parts = ax1.violinplot(data_cecal, showmeans=True)
    ax1.set_title("Cécal (J21)")
    ax1.set_xticks([1, 2])
    ax1.set_xticklabels(['Antibio', 'Placebo'])
    ax1.set_ylabel("Bactéries (Log10 CFU)")
    for pc in parts['bodies']:
        pc.set_facecolor('#D43F3A')
        pc.set_edgecolor('black')

    # Iléal 
    data_ileal = [ileal_abx, ileal_plb]
    parts2 = ax2.violinplot(data_ileal, showmeans=True)
    ax2.set_title("Iléal (J21)")
    ax2.set_xticks([1, 2])
    ax2.set_xticklabels(['Antibio', 'Placebo'])
    
    plt.savefig("graphique_violons.png")
    print("Graphique violons généré : graphique_violons.png")


# 1. Génération du fichier de données
generer_fichier_test("data_small.csv", n_souris=10) 

# 2. Analyse avec Pandas
analyser_donnees("data_small.csv")
