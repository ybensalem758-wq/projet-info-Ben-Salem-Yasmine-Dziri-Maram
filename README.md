import math
import matplotlib.pyplot as plt
import random


def generer_fichier_test(nom_fichier="data_small.csv", n_souris=20):
    """
    Génère un fichier CSV respectant la structure exacte demandée.
    Colonnes : 
    0:strain, 1:exp_id, 2:sample_type, 3:timepoint, 4:mouse_id, 
    5:treatment, 6:freq, 7:exp_day, 8:count, 9:age_days, 10:sex
    """
    print(f"Génération du fichier de test:{nom_fichier} ")
    f = open(nom_fichier,"w")
    # En-tête standard
    f.write("mouse_strain;experiment_id;sample_type;timepoint;mouse_id;treatment;"
            "frequency_live_bacteria;experimental_day;counts_live_bacteria;mouse_age_days;mouse_sex\n")
    
    jours_exp = [14, 15, 16, 17, 18, 19, 20, 21, 23, 25, 28]
    
    for i in range(1, n_souris + 1):
        # Formatage ID souris (ex: ABX001)
        prefix = "M"
        if i < 10: m_id = f"{prefix}00{i}"
        elif i < 100: m_id = f"{prefix}0{i}"
        else: m_id = f"{prefix}{i}"
            
        traitement = "ABX" if i <= n_souris / 2 else "PLB" # Moitié ABX, Moitié Placebo
        
        # 1. Données FÉCALES (Suivi temporel)
        for jour_vie in jours_exp:
            jour_exp = jour_vie - 14 # Jour relatif expérimental
            
            # Simulation biologique
            base = 10**8
            valeur = base
            if traitement == "ABX":
                if 14 <= jour_vie <= 20: valeur = base / 10000 # Chute
                else: valeur = base / 100 # Remontée partielle
            
            # Ajout de bruit aléatoire
            valeur = valeur * random.uniform(0.5, 1.5)
            
            # Écriture ligne
            ligne = f"C57BL6;EXP01;fecal;TP{jour_vie};{m_id};{traitement};0.9;{jour_exp};{int(valeur)};{jour_vie};M"
            f.write(ligne + "\n")

        # 2. Données CÉCALES et ILÉALES (Sacrifice à J21)
        # On ne génère ces données que pour le jour 21 (jour 7 exp)
        val_cecal = 10**7 if traitement == "PLB" else 10**4
        val_ileal = 10**6 if traitement == "PLB" else 10**3
        
        # Ajout variation
        val_cecal *= random.uniform(0.2, 5.0)
        val_ileal *= random.uniform(0.2, 5.0)

        f.write(f"C57BL6;EXP01;cecal;SACRIFICE;{m_id};{traitement};0.8;7;{int(val_cecal)};21;M\n")
        f.write(f"C57BL6;EXP01;ileal;SACRIFICE;{m_id};{traitement};0.8;7;{int(val_ileal)};21;M\n")
            
    f.close()
    print("Fichier généré avec succès.\n")


def analyser_donnees(nom_fichier_entree):
    print(f"Analyse du fichier : {nom_fichier_entree}")
    
    # Préparation des fichiers de sortie (Données filtrées)
    f_fecal_out = open("filtered_fecal.csv", "w")
    f_cecal_out = open("filtered_cecal.csv", "w")
    f_ileal_out = open("filtered_ileal.csv", "w")

    # Écriture des en-têtes filtrés (Filtrage Colonne)
    f_fecal_out.write("mouse_id;treatment;experimental_day;counts_live_bacteria\n")
    f_cecal_out.write("treatment;counts_live_bacteria\n") # Pas d'ID souris demandé
    f_ileal_out.write("treatment;counts_live_bacteria\n") # Pas d'ID souris demandé

    # Listes pour les graphiques Violon
    cecal_abx, cecal_plb = [], []
    ileal_abx, ileal_plb = [], []

    # Dictionnaire pour les graphiques Lignes
    # Structure : dico[mouse_id] = {'x': [], 'y': [], 'color': ''}
    donnees_courbes = {}

    # Ouverture du fichier source en lecture
    fd = open(nom_fichier_entree, "r")
    
    # Sauter la première ligne (en-tête)
    line = fd.readline()
    
    # Lire la première ligne de données
    line = fd.readline()
    
    while line != '':
        line = line.replace("\n", "")
        data = line.split(";")
        
        # Récupération des colonnes utiles via index
        # 2: sample_type, 4: mouse_id, 5: treatment, 7: exp_day, 8: count, 9: age_days
        
        sample_type = data[2]
        mouse_id = data[4]
        treatment = data[5]
        
        # Conversion sécurisée des chiffres
        try:
            exp_day = float(data[7])
            raw_count = float(data[8])
            age_days = int(data[9])
            log_count = math.log10(raw_count) if raw_count > 0 else 0
        except ValueError:
            line = fd.readline()
            continue

        # CAS 1 : DONNÉES FÉCALES (Graphique Lignes)
        # Filtrage Ligne : sample_type == 'fecal'
        if sample_type == 'fecal':
            # Filtrage Colonne pour CSV : ID, Traitement, Jour, Quantité
            ligne_csv = f"{mouse_id};{treatment};{exp_day};{raw_count}\n"
            f_fecal_out.write(ligne_csv)

            # Stockage en mémoire pour le graphique
            if mouse_id not in donnees_courbes:
                color = 'red' if treatment == 'ABX' else 'blue'
                donnees_courbes[mouse_id] = {'x': [], 'y': [], 'c': color}
            
            donnees_courbes[mouse_id]['x'].append(exp_day)
            donnees_courbes[mouse_id]['y'].append(log_count)

        # CAS 2 : DONNÉES CÉCALES (Graphique Violon)
        # Filtrage Ligne : sample_type == 'cecal' ET jour 21 (sacrifice)
        elif sample_type == 'cecal' and age_days == 21:
            # Filtrage Colonne pour CSV : Traitement, Quantité (Plus d'ID)
            ligne_csv = f"{treatment};{raw_count}\n"
            f_cecal_out.write(ligne_csv)

            # Ajout aux listes pour le graphique
            if treatment == 'ABX':
                cecal_abx.append(log_count)
            else:
                cecal_plb.append(log_count)

        # CAS 3 : DONNÉES ILÉALES (Graphique Violon)
        # Filtrage Ligne : sample_type == 'ileal' ET jour 21 (sacrifice)
        elif sample_type == 'ileal' and age_days == 21:
            # Filtrage Colonne pour CSV : Traitement, Quantité
            ligne_csv = f"{treatment};{raw_count}\n"
            f_ileal_out.write(ligne_csv)

            if treatment == 'ABX':
                ileal_abx.append(log_count)
            else:
                ileal_plb.append(log_count)

        # Lire la ligne suivante
        line = fd.readline()

    # Fermeture des fichiers
    fd.close()
    f_fecal_out.close()
    f_cecal_out.close()
    f_ileal_out.close()
    print("Fichiers CSV filtrés générés (filtered_fecal.csv, etc.)")
    
    
    # Graphique 1 : Courbes (Fécal)
    plt.figure(figsize=(10, 6))
    
    # On parcourt le dictionnaire rempli précédemment
    # Cela gère automatiquement un nombre indéfini de souris
    for m_id in donnees_courbes:
        d = donnees_courbes[m_id]
        # Tri des points par jour pour tracer une ligne propre
        points = sorted(zip(d['x'], d['y']))
        x_sorted = [p[0] for p in points]
        y_sorted = [p[1] for p in points]
        
        plt.plot(x_sorted, y_sorted, color=d['c'], alpha=0.6, linewidth=1.5)

    # Légende manuelle pour éviter d'avoir 50 fois "ABX" dans la légende
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

    #Graphique 2 : Violons (Cécal & Iléal)
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 6))

    # Cécal 
    data_cecal = [cecal_abx, cecal_plb]
    parts = ax1.violinplot(data_cecal, showmeans=True)
    ax1.set_title("Cécal (J21)")
    ax1.set_xticks([1, 2])
    ax1.set_xticklabels(['Antibio', 'Placebo'])
    ax1.set_ylabel("Bactéries (Log10 CFU)")
    
    # Colorier les violons pour différencier
    for pc in parts['bodies']:
        pc.set_facecolor('#D43F3A') # Rougeatre
        pc.set_edgecolor('black')

    # Iléal 
    data_ileal = [ileal_abx, ileal_plb]
    parts2 = ax2.violinplot(data_ileal, showmeans=True)
    ax2.set_title("Iléal (J21)")
    ax2.set_xticks([1, 2])
    ax2.set_xticklabels(['Antibio', 'Placebo'])
    
    plt.savefig("graphique_violons.png")
    print("Graphique violons généré : graphique_violons.png")

# 1. Je crée le fichier de données (comme si je le téléchargeais du cours)
generer_fichier_test("data_small.csv", n_souris=10) 

# 2. Je lance l'analyse
analyser_donnees("data_small.csv")

# Note : Pour tester "huge", il suffit de changer n_souris=500 dans la génération
# et de rappeler analyser_donnees.
