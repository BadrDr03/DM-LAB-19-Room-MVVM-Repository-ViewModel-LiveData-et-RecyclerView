# DM-LAB-19-Room-MVVM-Repository-ViewModel-LiveData-et-RecyclerView

Compétences visées
Comprendre la différence entre :
une application codée “directement dans l’Activity”
une application structurée selon MVVM
Comprendre le rôle exact de chaque couche :
Entity : structure des données persistées
DAO : interface d’accès aux données
RoomDatabase : point central de la base SQLite
Repository : couche intermédiaire entre données et ViewModel
ViewModel : logique de présentation et conservation de l’état de l’écran
LiveData : observation automatique et respect du cycle de vie
RecyclerView : affichage performant d’une collection de données
Comprendre pourquoi il ne faut pas exécuter les opérations Room sur le thread principal. Room est conçu pour structurer l’accès à SQLite, mais les opérations potentiellement bloquantes doivent rester hors du thread UI pour préserver la fluidité de l’application. que ViewModel résout réellement :
lors d’une rotation de l’écran
lors d’un changement de configuration
lors d’une recréation de l’Activity
Comprendre aussi sa limite : ViewModel aide pour les changements de configuration, mais il ne remplace pas totalement la gestion du process death ; pour cela, il existe notamment le module Saved State for ViewModel. at attendu
L’application finale contient :
un champ pour le titre
un champ pour la description
un bouton d’ajout
une liste des notes existantes
une suppression par clic long
une persistance locale via Room



Architecture globale du projet
Avant de coder, il faut comprendre l’architecture.
![Import OVA](https://github.com/user-attachments/assets/dee51e76-5544-4b1a-a591-4e752c857c9d)

Cette architecture suit le modèle MVVM et organise clairement la circulation des données entre l’interface utilisateur, la logique de présentation et la couche de persistance. Dans le sens descendant, MainActivity joue le rôle de vue : elle récupère les actions de l’utilisateur et délègue la logique au NoteViewModel. Le NoteViewModel ne dialogue pas directement avec la base ; il passe par le NoteRepository, qui centralise l’accès aux données. Ce repository utilise ensuite NoteDao, l’interface d’accès à la base Room/SQLite, pour exécuter les opérations de lecture, d’insertion ou de suppression.
Dans le sens remontant, les données stockées dans SQLite/Room sont exposées par le DAO sous forme de LiveData<List<Note>>. Cette donnée observable est transmise au Repository, puis au ViewModel, avant d’être observée par l’Activity. Dès qu’un changement survient dans la base, l’interface est automatiquement mise à jour, notamment dans le RecyclerView, sans avoir besoin de recharger manuellement les données.
Cette séparation apporte plusieurs avantages. Elle améliore d’abord la lisibilité, car chaque classe possède une responsabilité bien définie. Elle renforce ensuite la testabilité, puisque la logique métier et l’accès aux données sont isolés de l’interface graphique. Enfin, elle facilite la maintenance et l’évolution de l’application, car il devient plus simple d’ajouter de nouvelles fonctionnalités comme la recherche, la mise à jour ou la synchronisation avec une API distante. C’est précisément pour ces raisons que cette organisation est considérée comme une architecture moderne et robuste pour les applications Android.

Étape 1 — Création du projet et dépendances
Créer un projet :
Empty Views Activity
Nom : RoomMVVMDemo
Langage : Java
Min SDK : 24
Dépendances recommandées
Comme les versions évoluent, il est préférable d’utiliser des versions stables vérifiées. La page officielle AndroidX indique actuellement Room 2.8.4 comme stable et Lifecycle 2.9.3 comme version stable récente visible sur sa page de release notes. radle.ktsoubuild.gradle`, adapter selon le type du projet.
Version Groovy :
def room_version = "2.8.4"
def lifecycle_version = "2.9.3"

implementation "androidx.room:room-runtime:$room_version"
annotationProcessor "androidx.room:room-compiler:$room_version"
implementation "androidx.room:room-livedata:$room_version"

implementation "androidx.lifecycle:lifecycle-viewmodel:$lifecycle_version"
implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"

implementation "androidx.recyclerview:recyclerview:1.4.0"
implementation "androidx.cardview:cardview:1.0.0"
Explication des dépendances
room-runtime contient le moteur principal de Room.
room-compiler génère automatiquement le code nécessaire à partir des annotations @Entity, @Dao, @Database.
room-livedata permet de retourner directement un LiveData depuis le DAO.
lifecycle-viewmodel fournit ViewModel et AndroidViewModel.
lifecycle-livedata fournit LiveData et ses mécanismes d’observation.
recyclerview assure l’affichage performant des listes.
cardview ajoute un rendu plus lisible pour chaque note.Étape 2 — Organisation propre des packages
Créer une structure plus professionnelle :
com.example.roommvvmdemo
│
├── data
│   ├── local
│   │   ├── Note.java
│   │   ├── NoteDao.java
│   │   └── NoteDatabase.java
│   └── NoteRepository.java
│
├── ui
│   ├── MainActivity.java
│   └── NoteAdapter.java
│
└── viewmodel
    └── NoteViewModel.java
Cette séparation rend le projet beaucoup plus clair.

Étape 3 — Créer l’Entity
Créer Note.java dans data/local.
package com.example.roommvvmdemo.data.local;

import androidx.room.Entity;
import androidx.room.PrimaryKey;

@Entity(tableName = "notes_table")
public class Note {

    @PrimaryKey(autoGenerate = true)
    private int id;

    private String title;
    private String description;

    public Note(String title, String description) {
        this.title = title;
        this.description = description;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    } 

    public String getTitle() {
        return title;
    }

    public String getDescription() {
        return description;
    }
}
Explication détaillée
@Entity(tableName = "notes_table")
indique à Room que cette classe représente une table SQLite.
@PrimaryKey(autoGenerate = true)
demande à Room de générer automatiquement l’identifiant.
La classe contient ici trois colonnes :
id
title
description
Pourquoi le constructeur ne contient pas id ?
Parce que id est généré automatiquement par la base au moment de l’insertion.
Bonne pratique
Dans un projet réel, il serait possible d’ajouter :
une date de création
une date de modification
une priorité
un statut “archivée”
une validation des champs

Étape 4 — Créer le DAO
Créer NoteDao.java.
package com.example.roommvvmdemo.data.local;

import androidx.lifecycle.LiveData;
import androidx.room.Dao;
import androidx.room.Delete;
import androidx.room.Insert;
import androidx.room.Query;

import java.util.List;

@Dao
public interface NoteDao {

    @Insert
    void insert(Note note);

    @Delete
    void delete(Note note);

    @Query("DELETE FROM notes_table")
    void deleteAllNotes();

    @Query("SELECT * FROM notes_table ORDER BY id DESC")
    LiveData<List<Note>> getAllNotes();
}
Explication détaillée
@Dao
signale que cette interface regroupe les opérations d’accès aux données.
@Insert
indique une insertion.
@Delete
supprime l’objet passé en paramètre.
@Query("SELECT * FROM notes_table ORDER BY id DESC")
retourne toutes les notes, en commençant par la plus récente.
Le fait de retourner LiveData<List<Note>> est très important.
La documentation Android explique que LiveData peut être observé, et que cette observation respecte le cycle de vie du composant UI. Cela évite des mises à jour inutiles lorsque l’écran n’est plus actif. oom + LiveData est puissant ?
Quand les données changent dans la base, Room réémet automatiquement la nouvelle liste au LiveData. L’Activity n’a donc pas besoin d’aller relire manuellement la base après chaque insertionÉtape 5 — Créer la base Room
Créer NoteDatabase.java.
package com.example.roommvvmdemo.data.local;

import android.content.Context;

import androidx.room.Database;
import androidx.room.Room;
import androidx.room.RoomDatabase;

@Database(entities = {Note.class}, version = 1, exportSchema = false)
public abstract class NoteDatabase extends RoomDatabase {

    public abstract NoteDao noteDao();

    private static volatile NoteDatabase instance;

    public static NoteDatabase getInstance(Context context) {
        if (instance == null) {
            synchronized (NoteDatabase.class) {
                if (instance == null) {
                    instance = Room.databaseBuilder(
                                    context.getApplicationContext(),
                                    NoteDatabase.class,
                                    "notes_database"
                            )
                            .fallbackToDestructiveMigration()
                            .build();
                }
            }
        }
        return instance;
    }
}
Explication détaillée
@Database(...)
déclare la base Room.
entities = {Note.class}
la base contient ici une seule table.
version = 1
version actuelle du schéma.
exportSchema = false
évite l’export du schéma dans ce lab.
Pourquoi abstract ?
Room génère l’implémentation réelle à la compilation.
Pourquoi volatile ?
Le mot-clé volatile garantit une meilleure visibilité des changements entre threads.
Pourquoi synchronized ?
Pour empêcher la création simultanée de plusieurs instances de la base.


À propos de fallbackToDestructiveMigration()
Cette méthode est acceptable dans un laboratoire ou en phase de développement rapide. En revanche, en production, elle peut détruire les données si le schéma change. Dans une vraie application, il faut généralement définir des migrations explicites. La documentation Room insiste sur la gestion structurée du schéma et des migrations.

Étape 6 — Créer le Repository
Le Repository joue le rôle d’intermédiaire entre le ViewModel et la base de données.
Son objectif est de centraliser l’accès aux données et de cacher les détails d’implémentation liés à Room.
Pourquoi cette étape est importante
Sans Repository, le ViewModel devrait communiquer directement avec le DAO.
Cela fonctionnerait dans un petit projet, mais l’architecture deviendrait vite difficile à maintenir.
Le Repository permet donc :
d’isoler la logique d’accès aux données
de préparer facilement une évolution vers une source distante comme Retrofit
de garder un ViewModel plus simple et plus propre
Créer la classe NoteRepository
Créer le fichier NoteRepository.java.
package com.example.roommvvmdemo.data;

import android.app.Application;

import androidx.lifecycle.LiveData;

import com.example.roommvvmdemo.data.local.Note;
import com.example.roommvvmdemo.data.local.NoteDao;
import com.example.roommvvmdemo.data.local.NoteDatabase;

import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class NoteRepository {

    private final NoteDao noteDao;
    private final LiveData<List<Note>> allNotes;
    private final ExecutorService executorService;

    public NoteRepository(Application application) {
        NoteDatabase database = NoteDatabase.getInstance(application);
        noteDao = database.noteDao();
        allNotes = noteDao.getAllNotes();
        executorService = Executors.newSingleThreadExecutor();
    }

    public void insert(Note note) {
        executorService.execute(() -> noteDao.insert(note));
    }

    public void delete(Note note) {
        executorService.execute(() -> noteDao.delete(note));
    }

    public void deleteAllNotes() {
        executorService.execute(noteDao::deleteAllNotes);
    }

    public LiveData<List<Note>> getAllNotes() {
        return allNotes;
    }
}
Explication ligne par ligne
private final NoteDao noteDao;
contient une référence vers l’interface DAO.
private final LiveData<List<Note>> allNotes;
stocke la liste observable des notes.
private final ExecutorService executorService;
permet d’exécuter les opérations d’écriture dans un thread secondaire.
Dans le constructeur :
NoteDatabase database = NoteDatabase.getInstance(application);
récupère l’instance unique de la base.
noteDao = database.noteDao();
permet d’obtenir le DAO.
allNotes = noteDao.getAllNotes();
récupère les notes sous forme de LiveData.
executorService = Executors.newSingleThreadExecutor();
crée un seul thread d’arrière-plan dédié aux opérations de base de données.
Pourquoi utiliser ExecutorService ?
Les opérations d’insertion et de suppression ne doivent pas être exécutées sur le thread principal, sinon l’interface risque de se figer.
Le Repository garantit ici que les opérations sont lancées en arrière-plan.
