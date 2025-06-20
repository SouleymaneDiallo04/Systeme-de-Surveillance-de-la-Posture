----
II.Approche Technique
=====================


----

Prétraitement des Données
-------------------------

----

Pour notre projet, nous avons adapté le traitement des données selon le modèle :  
- **Détection de chute**  
- **Prédiction de chute**  
- **Détection de somnolence**  

Dans cette section, nous détaillons l’approche spécifique pour le modèle de détection de somnolence.

----

Chargement des données
----------------------

----

Nous téléchargeons le dataset directement depuis Kaggle à l’aide de ``kagglehub`` :

.. code-block:: python

    import kagglehub

    # Télécharger la dernière version du dataset
    path = kagglehub.dataset_download("rakibuleceruet/drowsiness-prediction-dataset")

    print("Path to dataset files:", path)
    
    
----
    
    
Utilisation de Mediapipe pour les landmarks faciaux
---------------------------------------------------

----

Nous utilisons ``mediapipe`` pour extraire les points de repère (landmarks) du visage :  

.. code-block:: python

    import mediapipe as mp

    mp_face_mesh = mp.solutions.face_mesh
    mp_drawing  = mp.solutions.drawing_utils

    face_mesh = mp_face_mesh.FaceMesh(
        min_detection_confidence=0.3,
        min_tracking_confidence=0.8
    )

----


Calcul des caractéristiques (features) visuelles
------------------------------------------------

----

Les fonctions suivantes permettent de calculer :
- ``eye_aspect_ratio`` : ouverture des yeux
- ``mouth_feature`` : ouverture de la bouche

.. code-block:: python

    import numpy as np

    #Fonction pour calculer la distance euclidienne entre deux points
    def distance(p1, p2):
        return (((p1[:2] - p2[:2])**2).sum())**0.5

    # Ratio pour les yeux
    def eye_aspect_ratio(landmarks, eye):
        N1 = distance(landmarks[eye[1][0]], landmarks[eye[1][1]])
        N2 = distance(landmarks[eye[2][0]], landmarks[eye[2][1]])
        N3 = distance(landmarks[eye[3][0]], landmarks[eye[3][1]])
        D = distance(landmarks[eye[0][0]], landmarks[eye[0][1]])
        return (N1 + N2 + N3) / (3 * D)

    # Moyenne des deux yeux
    def eye_feature(landmarks):
        return (eye_aspect_ratio(landmarks, left_eye) + eye_aspect_ratio(landmarks, right_eye)) / 2

    # Ratio pour la bouche
    def mouth_feature(landmarks):
        N1 = distance(landmarks[mouth[1][0]], landmarks[mouth[1][1]])
        N2 = distance(landmarks[mouth[2][0]], landmarks[mouth[2][1]])
        N3 = distance(landmarks[mouth[3][0]], landmarks[mouth[3][1]])
        D = distance(landmarks[mouth[0][0]], landmarks[mouth[0][1]])
        return (N1 + N2 + N3) / (3 * D)
        
----

Extraction des caractéristiques pour les images somnolentes
-----------------------------------------------------------

----

.. code-block:: python

    import os

    somnol_feats = []
    somnol_path = '/content/drowsiness_dataset/0 FaceImages/Fatigue Subjects'

    somnol_list = os.listdir(somnol_path)
    print(f"Nombre d'images somnolentes : {len(somnol_list)}")


Extraction des caractéristiques pour les images non somnolentes


Pour chaque image active, nous appliquons :
- Flip horizontal
- Conversion RGB
- Détection des landmarks
- Extraction des features ``ear`` (yeux) et ``mar`` (bouche)

.. code-block:: python

    import cv2
    import pickle

    active_feats = []
    active_path = '/content/drowsiness_dataset/0 FaceImages/Active Subjects'
    active_list = os.listdir(active_path)

    for name in active_list:
        image_path = os.path.join(active_path, name)
        image = cv2.imread(image_path)

        if image is None:
            print(f"Image non lue : {image_path}")
            continue

        # Flip horizontal & conversion RGB
        image_rgb = cv2.cvtColor(cv2.flip(image, 1), cv2.COLOR_BGR2RGB)

        # Détection des landmarks
        results = face_mesh.process(image_rgb)

        if results.multi_face_landmarks:
            landmarks_positions = []
            for lm in results.multi_face_landmarks[0].landmark:
                landmarks_positions.append([lm.x, lm.y, lm.z])

            # Dénormalisation
            landmarks_positions = np.array(landmarks_positions)
            landmarks_positions[:, 0] *= image.shape[1]  # largeur
            landmarks_positions[:, 1] *= image.shape[0]  # hauteur

            ear = eye_feature(landmarks_positions)
            mar = mouth_feature(landmarks_positions)
            active_feats.append((ear, mar))

    # Sauvegarde des features
    active_feats = np.array(active_feats)
    os.makedirs("./feats", exist_ok=True)
    with open("./feats/mp_active_feats.pkl", "wb") as f:
        pickle.dump(active_feats, f)
        
----

Préparation des jeux de données
 -------------------------------
----

On crée les labels et on divise en jeux d’entraînement/test :  

.. code-block:: python

    import sklearn

    np.random.seed(42)

    somnol_labs = np.zeros(len(somnol_feats))
    active_labs = np.ones(len(active_feats))

    X = np.vstack((somnol_feats, active_feats))
    y = np.concatenate((somnol_labs, active_labs))

    X_train, X_test, y_train, y_test = sklearn.model_selection.train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
----
    
Fonction pour visualisation et redimensionnement des images
 -----------------------------------------------------------
 
 ----

La fonction ``draw`` permet :  
- De dessiner les landmarks (tessellation, yeux, bouche)
- De sauvegarder les images annotées
- De redimensionner les images pour le modèle

.. code-block:: python

    IMG_SIZE = 145

    def draw(
        *, img_dt, cat, image_name,
        img_eye_lmks=None, img_eye_lmks_chosen=None,
        face_landmarks=None,
        ts_thickness=1, ts_circle_radius=2, lmk_circle_radius=3
    ):
        imgH, imgW = img_dt.shape[:2]

        # Pour dessiner la tessellation
        image_drawing_tool = img_dt
        image_eye_lmks = img_dt.copy() if img_eye_lmks is None else img_eye_lmks
        img_eye_lmks_chosen = img_dt.copy() if img_eye_lmks_chosen is None else img_eye_lmks_chosen

        connections_drawing_spec = mp_drawing.DrawingSpec(
            thickness=ts_thickness,
            circle_radius=ts_circle_radius,
            color=(255, 255, 255)
        )

        # Dessin de la tessellation faciale
        mp_drawing.draw_landmarks(
            image=image_drawing_tool,
            landmark_list=face_landmarks,
            connections=mp_face_mesh.FACEMESH_TESSELATION,
            landmark_drawing_spec=None,
            connection_drawing_spec=connections_drawing_spec,
        )

        # Dessin des points choisis
        landmarks = face_landmarks.landmark
        for landmark_idx, landmark in enumerate(landmarks):
            if landmark_idx in all_idxs:
                pred_cord = denormalize_coordinates(landmark.x, landmark.y, imgW, imgH)
                cv2.circle(image_eye_lmks, pred_cord, lmk_circle_radius, (255, 255, 255), -1)

            if landmark_idx in all_chosen_idxs:
                pred_cord = denormalize_coordinates(landmark.x, landmark.y, imgW, imgH)
                cv2.circle(img_eye_lmks_chosen, pred_cord, lmk_circle_radius, (255, 255, 255), -1)

        # Sauvegarde de l’image annotée
        save_path = os.path.join(
            '/content/drowsiness_dataset/0 FaceImages/Fatigue Subjects' if cat == 'Fatigue Subjects'
            else '/content/drowsiness_dataset/0 FaceImages/Active Subjects',
            image_name
        )
        cv2.imwrite(save_path, image_drawing_tool)

        # Redimensionnement pour le modèle
        resized_array = cv2.resize(image_drawing_tool, (IMG_SIZE, IMG_SIZE))
        return resized_array
        
----

Traitement d’image pour le modèle de prédiction
===============================================

----

Ce module de préparation des données vise à transformer les vidéos en **séquences d’images** normalisées, exploitables par un modèle de prédiction de chutes basé sur des réseaux de neurones (CNN + LSTM).

----
Objectif
--------

----

Prétraiter les vidéos afin de :

- Extraire un nombre fixe de **30 images par vidéo**.
- Redimensionner chaque image à **128x128 pixels**.
- Étiqueter les vidéos selon leur nature : **chute (0)** ou **mouvement normal (1)**.
- Créer un ensemble d’apprentissage et de test au format attendu par un modèle Keras.

----
Paramètres
----------
----

- ``SEQUENCE_LENGTH = 30`` : nombre d’images extraites par vidéo.
- ``IMG_SIZE = (128, 128)`` : taille à laquelle chaque image est redimensionnée.
- ``TEST_RATIO = 0.05`` : proportion de données utilisées pour le test.
- ``VIDEO_PATH_CHUTES`` : dossier contenant les vidéos de chutes.
- ``VIDEO_PATH_NORMAUX`` : dossier contenant les vidéos normales.

----

Étapes du traitement
--------------------

----

1. **Extraction des images** :
   - Chaque vidéo est lue image par image.
   - Les images sont converties en **RGB** puis redimensionnées à ``128x128``.
   - Seules les vidéos comportant **au moins 30 images** sont retenues.
   - Le tout est stocké sous forme de tableau NumPy.

2. **Étiquetage** :
   - Les vidéos de chutes reçoivent le label ``0``.
   - Les vidéos normales reçoivent le label ``1``.

3. **Préparation des données** :
   - Les données (``X``) sont converties en tableau NumPy de forme ``(N, 30, 128, 128, 3)``.
   - Les étiquettes (``y``) sont transformées en **one-hot encoding** via ``to_categorical``.

4. **Découpage en train/test** :
   - Utilisation de ``train_test_split`` avec stratification selon les classes.
   - Ratio test fixé à 5 %.

Exemple d’exécution
-------------------

À la fin de l’exécution du script :

.. code-block:: bash

   Données préparées : (95, 30, 128, 128, 3) (train), (5, 30, 128, 128, 3) (test)

Cela signifie que 100 vidéos ont été traitées, et 95 sont utilisées pour l’entraînement.

----

Code source
-----------
----

.. code-block:: python

   import cv2
   import os
   import numpy as np
   from sklearn.model_selection import train_test_split
   from keras.utils import to_categorical

   SEQUENCE_LENGTH = 30
   IMG_SIZE = (128, 128)
   TEST_RATIO = 0.05

   VIDEO_PATH_CHUTES = r"../data/dataAlgorithme1/VideoChute"
   VIDEO_PATH_NORMAUX = r"../data/dataAlgorithme1/videoNormal"

   def extract_frames(video_path, num_frames=SEQUENCE_LENGTH):
       cap = cv2.VideoCapture(video_path)
       frames = []
       while len(frames) < num_frames and cap.isOpened():
           ret, frame = cap.read()
           if not ret:
               break
           frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
           frame = cv2.resize(frame, IMG_SIZE)
           frames.append(frame)
       cap.release()
       return np.array(frames[:num_frames])

   X, y = [], []

   for video_file in os.listdir(VIDEO_PATH_CHUTES):
       video_path = os.path.join(VIDEO_PATH_CHUTES, video_file)
       sequence = extract_frames(video_path)
       if len(sequence) == SEQUENCE_LENGTH:
           X.append(sequence)
           y.append(0)

   for video_file in os.listdir(VIDEO_PATH_NORMAUX):
       video_path = os.path.join(VIDEO_PATH_NORMAUX, video_file)
       sequence = extract_frames(video_path)
       if len(sequence) == SEQUENCE_LENGTH:
           X.append(sequence)
           y.append(1)

   X = np.array(X)
   y = to_categorical(np.array(y))
   X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=TEST_RATIO, stratify=y)
   print(f"Données préparées : {X_train.shape} (train), {X_test.shape} (test)")

----

