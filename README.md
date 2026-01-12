# Projet-ORAL
# Projet Instagram du Groupe AGOSSOU-AHLEGNAN-BEN HIM
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
INSTAGRAM IMPACT PROJECT: THE ULTIMATE ACTIVE USER TOOL
- Safe Save: Forces results to save in the script's folder.
- Deep Search: Finds data anywhere in the home directory.
- Auto-Detection: Identifies the active user automatically.
"""

import json
import os
import csv
import sys
from datetime import datetime
from collections import Counter

# Silence Pylance/Import warnings
try:
    import matplotlib.pyplot as plt
    HAS_MATPLOTLIB = True
except (ImportError, ModuleNotFoundError):
    HAS_MATPLOTLIB = False

class InstagramDetectiveTool:
    def __init__(self):
        self.data_path = None
        self.owner_name = "User"
        # Force saving to the folder where the script lives
        self.output_dir = os.path.dirname(os.path.abspath(__file__))

    # --- 1. SMART DATA DETECTION ---
    def find_data_anywhere(self):
        """Recursively searches for unzipped Instagram data"""
        print("🔍 Searching your computer for Instagram data...")
        
        quick_spots = [os.path.expanduser("~/Downloads"), os.path.expanduser("~/Desktop"), os.getcwd()]
        
        valid_folders = []
        for spot in quick_spots:
            if not os.path.exists(spot): continue
            try:
                for item in os.listdir(spot):
                    path = os.path.join(spot, item)
                    if os.path.isdir(path) and "instagram" in item.lower():
                        if os.path.exists(os.path.join(path, 'messages')) or \
                           os.path.exists(os.path.join(path, 'your_instagram_activity')):
                            valid_folders.append(path)
            except: continue
            
        if valid_folders:
            self.data_path = max(valid_folders, key=os.path.getmtime)
            print(f"✅ Found data: {os.path.basename(self.data_path)}")
            return True

        print("🕵️  Deep scanning home directory (this may take a moment)...")
        home = os.path.expanduser("~")
        for root, dirs, _ in os.walk(home):
            dirs[:] = [d for d in dirs if not d.startswith('.') and d not in 
                      ['Library', 'Applications', 'Public', 'Movies', 'Music']]
            for d in dirs:
                if "instagram" in d.lower():
                    path = os.path.join(root, d)
                    if os.path.exists(os.path.join(path, 'messages')):
                        self.data_path = path
                        print(f"🎯 Found: {path}")
                        return True
        return False

    # --- 2. AUTO-OWNER DETECTION ---
    def detect_owner(self):
        all_senders = []
        for root, _, files in os.walk(self.data_path):
            if 'inbox' in root.lower():
                for file in files:
                    if file.endswith('.json') and len(all_senders) < 300:
                        try:
                            with open(os.path.join(root, file), 'r', encoding='utf-8') as f:
                                data = json.load(f)
                                for msg in data.get('messages', []):
                                    if msg.get('sender_name'): all_senders.append(msg.get('sender_name'))
                        except: continue
        self.owner_name = Counter(all_senders).most_common(1)[0][0] if all_senders else "User"
        print(f"👤 Account owner: {self.owner_name}")

    # --- 3. DATA EXTRACTOR ---
    def extract_dates(self, filepath, data_type):
        if not filepath or not os.path.isfile(filepath): return []
        dates = []
        try:
            with open(filepath, 'r', encoding='utf-8') as f:
                data = json.load(f)
                
                def get_ts(item):
                    if not isinstance(item, dict): return None
                    if 'string_map_data' in item and 'Time' in item['string_map_data']:
                        return item['string_map_data']['Time'].get('timestamp')
                    if 'creation_timestamp' in item: return item['creation_timestamp']
                    if 'string_list_data' in item and isinstance(item['string_list_data'], list):
                        if item['string_list_data']: return item['string_list_data'][0].get('timestamp')
                    return item.get('timestamp') or (item.get('timestamp_ms', 0) / 1000)

                items = []
                if data_type == "comments" and isinstance(data, dict) and 'string_map_data' in data:
                    items = [data]
                elif isinstance(data, list): items = data
                else:
                    for k in ['likes_media_likes', 'ig_stories', 'messages', 'comments']:
                        if k in data and isinstance(data[k], list): items = data[k]; break

                for i in items:
                    if data_type == "messages" and i.get('sender_name') != self.owner_name: continue
                    ts = get_ts(i)
                    if ts and 1262304000 < ts < 2524608000: dates.append(datetime.fromtimestamp(ts))
        except: pass
        return dates

    # --- 4. EXECUTION ---
    def run(self):
        if not self.find_data_anywhere():
            print("❌ Instagram data not found.")
            return
        
        self.detect_owner()
        likes, comments, stories, messages = [], [], [], []
        print("\n📊 Analyzing activity...")
        
        for root, _, files in os.walk(self.data_path):
            for file in files:
                if not file.endswith('.json'): continue
                p, f = os.path.join(root, file), file.lower()
                if 'liked_posts' in f or ('likes' in f and 'comment' not in f): likes.extend(self.extract_dates(p, "likes"))
                elif 'comments' in f: comments.extend(self.extract_dates(p, "comments"))
                elif 'stories' in f and 'interactions' not in f: stories.extend(self.extract_dates(p, "stories"))
                elif 'message' in f and 'inbox' in root.lower(): messages.extend(self.extract_dates(p, "messages"))

        self.generate_report(likes, comments, messages, stories)

    def generate_report(self, likes, comments, messages, stories):
        all_act = likes + comments + messages + stories
        
        print("\n" + "="*65)
        print(f"🚀 ANALYSIS COMPLETE: {self.owner_name.upper()}")
        print("="*65)
        print(f"📊 COUNTS: ❤️ {len(likes)} | 💬 {len(comments)} | 📩 {len(messages)} | 🎞️ {len(stories)}")
        
        if all_act:
            # Save CSV to script folder
            csv_path = os.path.join(self.output_dir, "active_user_results.csv")
            with open(csv_path, "w", newline="", encoding="utf-8") as f:
                w = csv.writer(f)
                w.writerow(["Action", "Date", "Hour"])
                for label, d_list in [("Like", likes), ("Comment", comments), ("Message", messages), ("Story", stories)]:
                    for date in d_list: w.writerow([label, date.date(), date.hour])
            print(f"\n📁 File saved here: {csv_path}")

            # Timeline
            month_counts = Counter([d.strftime('%Y-%m') for d in all_act])
            scale = max(1, max(month_counts.values()) // 20)
            print("\n📅 ACTIVITY TREND:")
            for month in sorted(month_counts.keys()):
                bar = '●' * (month_counts[month] // scale)
                print(f"   {month}: {bar} ({month_counts[month]})")
            
            if HAS_MATPLOTLIB:
                self.plot(month_counts)

    def plot(self, month_counts):
        months = sorted(month_counts.keys())
        counts = [month_counts[m] for m in months]
        plt.figure(figsize=(10, 5))
        plt.plot(months, counts, marker='o', color='purple', linewidth=2)
        plt.title(f'Repartition volume activité: {self.owner_name}')
        plt.xticks(rotation=45)
        plt.tight_layout()
        graph_path = os.path.join(self.output_dir, "impact_trend.png")
        plt.savefig(graph_path)
        print(f"🖼️  Graph saved here: {graph_path}")

if __name__ == "__main__":
    InstagramDetectiveTool().run()

    # -*- coding: utf-8 -*-



import os
import json
import pandas as pd

## La fonction mise à jour 

def update_data(root_folder):
    dfs = {}
    for subdir, dirs, files in os.walk(root_folder):
        for file in files:
            if file.endswith(".json"):
                file_path = os.path.join(subdir, file)
                try:
                    with open(file_path, "r", encoding="utf-8") as f:
                        data = json.load(f)
                        df = pd.DataFrame(data)
                        # Convertir timestamp si présent
                        if "timestamp" in df.columns:
                            df["timestamp"] = pd.to_datetime(df["timestamp"], unit="s", errors="coerce")
                        dfs[file] = df
                        print(f"{file} chargé — {len(df)} lignes")
                except:
                    pass
    return dfs

# Automatisation 
def find_instagram_folder(start_path="."):
    for root, dirs, files in os.walk(start_path):
        if "your_instagram_activity" in dirs:
            return os.path.join(root, "your_instagram_activity")
    return None

root_folder = find_instagram_folder()

if root_folder is None:
    raise FileNotFoundError("Dossier 'your_instagram_activity' introuvable. Place le script dans le dossier de l'export Instagram.")
else:
    print("Dossier Instagram détecté :", root_folder)

dfs = update_data(root_folder)


print(os.path.exists(root_folder))
print(os.listdir(root_folder))



donnees_likes_publications = dfs.get("liked_posts.json")
donnees_likes_stories = dfs.get("story_likes.json")
donnees_likes_commentaires = dfs.get("liked_comments.json")
donnees_commentaires_publications = dfs.get("post_comments_1.json")
donnees_publications_enregistrees = dfs.get("saved_posts.json")



## EVOLUTION MENSUELLE DES LIKES


if donnees_likes_publications is not None and "timestamp" in donnees_likes_publications.columns:
    
    # Création d'une colonne mois
    donnees_likes_publications["mois"] = donnees_likes_publications["timestamp"].dt.to_period("M")
    
    evolution_mensuelle_likes = donnees_likes_publications.groupby("mois").size()

    print("\nEvolution des likes de publications par mois :")
    print(evolution_mensuelle_likes.tail())
    
    
##  MOMENTS D'ACTIVITE DANS LA JOURNEE


if donnees_likes_publications is not None and "timestamp" in donnees_likes_publications.columns:
    
    donnees_likes_publications["heure"] = donnees_likes_publications["timestamp"].dt.hour
    activite_par_heure = donnees_likes_publications.groupby("heure").size()

    print("\nRépartition des likes selon l'heure de la journée :")
    print(activite_par_heure)



##  PROPORTION DES TYPES D'ACTIVITES

# Préparations des données des likes pour l'analyse
likes_publications = dfs.get("liked_posts.json")
likes_stories = dfs.get("story_likes.json")
likes_commentaires = dfs.get("liked_comments.json")
commentaires_publications = dfs.get("post_comments_1.json")
publications_enregistrees = dfs.get("saved_posts.json")

# Comptage des lignes pour chaque activité
compte_activites = {
    "Likes de publications": 0 if likes_publications is None else len(likes_publications),
    "Likes de stories": 0 if likes_stories is None else len(likes_stories),
    "Likes de commentaires": 0 if likes_commentaires is None else len(likes_commentaires),
    "Commentaires": 0 if commentaires_publications is None else len(commentaires_publications),
    "Publications enregistrées": 0 if publications_enregistrees is None else len(publications_enregistrees)}

# Somme totale des actions
total_activites = sum(compte_activites.values())

# Calcul et affichage des proportions
print("\nProportion des différentes activités sur Instagram :")
if total_activites == 0:
    print("Aucune activité détectée")
else:
    for nom, valeur in compte_activites.items():
        proportion = (valeur / total_activites) * 100
        print(f"{nom} : {proportion:.2f}% ({valeur} actions)")
        
        
        # --- GRAPHIQUE PIE CHART (VERSION PLEINE) ---
    if HAS_MATPLOTLIB:
        # Filtrer les données pour ignorer les catégories à 0
        filtered_data = {k: v for k, v in compte_activites.items() if v > 0}
        
        if filtered_data:
            plt.figure(figsize=(10, 7))
            
            # Palette de couleurs style Instagram
            colors = ['#FF66B2', '#fcb045', '#5DD5FC', '#833ab4', '#fd1d1d']
            
            plt.pie(filtered_data.values(), 
                    labels=filtered_data.keys(), 
                    autopct='%1.1f%%', 
                    startangle=140, 
                    colors=colors[:len(filtered_data)],
                    wedgeprops={'edgecolor': 'white', 'linewidth': 1.5})
            
            plt.title("Répartition de ton activité Instagram", fontsize=14, fontweight='bold')
            plt.axis('equal') # Assure que le cercle est bien rond
            plt.tight_layout()
            plt.show()
        
      
import pandas as pd

# DataFrame de liked_posts
df_likes = dfs.get('liked_posts.json')

if df_likes is not None and 'likes_media_likes' in df_likes.columns:
    
    # Transformer les listes imbriquées en DataFrame
    df_exp = pd.json_normalize(df_likes['likes_media_likes'])
    
    # Créer une nouvelle colonne timestamp à partir de string_list_data
    # On suppose que chaque action a un champ 'timestamp' dans string_list_data
    timestamps = []
    for row in df_exp['string_list_data']:
        # row est une liste de dictionnaires
        if isinstance(row, list) and len(row) > 0:
            # On prend le premier dictionnaire qui a 'timestamp'
            t = next((item.get('timestamp') for item in row if 'timestamp' in item), None)
            timestamps.append(t)
        else:
            timestamps.append(None)
    
    df_exp['date_action'] = pd.to_datetime(timestamps, unit='s', errors='coerce')
    
    print(df_exp[['title', 'date_action']].head())
    
    #  Évolution journalière 
    df_exp['jour'] = df_exp['date_action'].dt.date
    freq_jour = df_exp.groupby('jour').size()
    print("\nÉvolution journalière des likes de publications :")
    print(freq_jour.head())
    
    # Évolution mensuelle
    df_exp['mois'] = df_exp['date_action'].dt.to_period('M')
    freq_mois = df_exp.groupby('mois').size()
    print("\nÉvolution mensuelle des likes de publications :")
    print(freq_mois.head())


      

import pandas as pd
import matplotlib.pyplot as plt

## Transormations des listes en dataframes
df_likes = dfs.get('liked_posts.json')

df_likes_exp = pd.json_normalize(df_likes['likes_media_likes'])

df_likes_exp['date_action'] = pd.to_datetime(
    df_likes_exp['string_list_data'].apply(lambda x: x[0]['timestamp']),
    unit='s')


# FRÉQUENCE JOURNALIÈRE 
df_likes_exp["jour"] = df_likes_exp["date_action"].dt.date
freq_par_jour = df_likes_exp["jour"].value_counts().sort_index()

print("\nÉvolution journalière des likes :")
print(freq_par_jour.head())

# JOUR LE PLUS ADDICTIF
max_day = freq_par_jour.idxmax()
max_value = freq_par_jour.max()

print("\n JOUR LE PLUS ACTIF ('Addiction Day')")
print(f" {max_day} avec {max_value} likes")

## Les courbes


## Evolution mensuelle

evolution_mois = df_likes_exp.groupby(df_likes_exp["date_action"].dt.to_period("M")).size()

plt.figure(figsize=(8,5))
evolution_mois.plot(kind='line', marker='o')
plt.title("Évolution mensuelle des likes de publications")
plt.xlabel("Mois")
plt.ylabel("Nombre de likes")
plt.grid(True)
plt.show()


## Top comptes interactions

top_accounts = df_likes_exp["title"].value_counts().head(10)
print(top_accounts)

top_accounts.plot(kind="bar", figsize=(8,5))
plt.title("Top 10 comptes avec lesquels j'intéragis le plus")
plt.ylabel("Nombre de likes")
plt.show()


# Profil utilisateur ANALYSE GLOBALE INSTAGRAM

import csv
from collections import Counter
import os

# CHARGEMENT AUTOMATIQUE DES STATS ACTIVES


def load_active_stats(csv_path):
    actions = []

    with open(csv_path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            actions.append(row["Action"])

    counts = Counter(actions)

    return {
        "likes": counts.get("Like", 0),
        "comments": counts.get("Comment", 0),
        "messages": counts.get("Message", 0),
        "stories": counts.get("Story", 0)
    }


# ANALYSE GLOBALE

def user_profile_summary(active_stats):
    total_actions = sum(active_stats.values())

    engagement_score = (
        active_stats["likes"] * 1 +
        active_stats["messages"] * 2 +
        active_stats["comments"] * 3
    )

    profile = {
        "total_actions": total_actions,
        "engagement_score": engagement_score,
        "dominant_behavior":
            "passive" if active_stats["likes"] > active_stats["comments"] else "active",
        "visibility_level":
            "low" if active_stats["comments"] < 5 else "medium"
    }

    return profile


# EXÉCUTION AUTOMATIQUE

CSV_PATH = os.path.join(os.path.dirname(__file__), "active_user_results.csv")

active_stats = load_active_stats(CSV_PATH)
profile = user_profile_summary(active_stats)

print("\n📌 PROFIL UTILISATEUR GLOBAL (AUTO)")
print("-" * 45)
for k, v in profile.items():
    print(f"{k} : {v}")
