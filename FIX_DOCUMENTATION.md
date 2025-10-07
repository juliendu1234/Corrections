# Fix for Video Recording and Photo Capture Issues

## Problèmes Identifiés

### 1. Enregistrement Vidéo
**Symptôme**: Le fichier .mp4 est créé dans Documents mais reste toujours à 0 octets.

**Cause**: Le code initialisait l'`AVAssetWriter` et démarrait la session d'écriture, mais ne transmettait jamais les frames vidéo décodées à l'enregistreur. Les frames n'étaient affichées qu'à l'écran mais jamais écrites dans le fichier.

### 2. Capture Photo
**Symptôme**: Aucune action ne se produit quand on appuie sur le bouton de capture photo (D-PAD RIGHT).

**Cause**: La fonction `capturePhoto()` imprimait seulement un message de log mais ne capturait et ne sauvegardait aucune image.

## Solutions Implémentées

### 1. Correction de l'Enregistrement Vidéo

**Fichier**: `Sources/VideoStreamHandler.swift`

#### Changements effectués:

1. **Ajout de variables de suivi**:
   ```swift
   private var recordingFrameCount: Int64 = 0
   private var latestImageBuffer: CVImageBuffer?
   ```

2. **Modification de la fonction `displayFrame()`**:
   - Sauvegarde la dernière frame dans `latestImageBuffer` pour la capture photo
   - Ajoute la logique d'écriture des frames quand l'enregistrement est actif:
     ```swift
     if isRecording, let input = videoWriterInput, let writer = videoWriter {
         if input.isReadyForMoreMediaData && writer.status == .writing {
             // Crée un sample buffer avec le bon timestamp
             // Écrit la frame dans le fichier vidéo
             input.append(recordingSample)
             recordingFrameCount += 1
         }
     }
     ```

3. **Amélioration du logging**:
   - Affiche le nombre total de frames enregistrées lors de l'arrêt de l'enregistrement

### 2. Implémentation de la Capture Photo

**Fichier**: `Sources/VideoStreamHandler.swift`

#### Nouvelle fonction `capturePhoto()`:

```swift
func capturePhoto() -> URL? {
    // 1. Récupère la dernière frame vidéo disponible
    guard let imageBuffer = latestImageBuffer else { return nil }
    
    // 2. Convertit CVImageBuffer -> CIImage -> CGImage
    let ciImage = CIImage(cvImageBuffer: imageBuffer)
    let context = CIContext()
    let cgImage = context.createCGImage(ciImage, from: ciImage.extent)
    
    // 3. Convertit en JPEG avec compression 90%
    let bitmapRep = NSBitmapImageRep(cgImage: cgImage)
    let jpegData = bitmapRep.representation(using: .jpeg, properties: [.compressionFactor: 0.9])
    
    // 4. Sauvegarde dans Documents avec horodatage
    let filename = "ARDrone_Photo_\(timestamp).jpg"
    try jpegData.write(to: photoURL)
    
    return photoURL
}
```

**Fichier**: `Sources/ARDroneController.swift`

#### Mise à jour de `capturePhoto()`:
- Utilise maintenant la nouvelle fonction de `VideoStreamHandler`
- Retourne l'URL de la photo capturée
- Affiche des messages de succès/échec appropriés

### 3. Permissions Info.plist

**Fichier**: `Info.plist`

Ajout des permissions nécessaires pour l'accès aux fichiers et médias:

```xml
<key>com.apple.security.files.user-selected.read-write</key>
<true/>
<key>com.apple.security.assets.movies.read-write</key>
<true/>
<key>com.apple.security.assets.pictures.read-write</key>
<true/>
```

## Détails Techniques

### Timestamp Vidéo
- Les frames sont enregistrées avec un timestamp basé sur `recordingFrameCount`
- Timescale: 30 (correspondant à 30 FPS)
- Chaque frame: `CMTime(value: recordingFrameCount, timescale: 30)`

### Format Photo
- Format: JPEG
- Qualité: 90% (compressionFactor: 0.9)
- Résolution: Identique au flux vidéo (1280x720)
- Nomenclature: `ARDrone_Photo_YYYYMMdd_HHmmss.jpg`

### Format Vidéo
- Codec: H.264
- Résolution: 1280x720
- Bitrate: 2 Mbps
- FPS: 30
- Nomenclature: `ARDrone_YYYYMMdd_HHmmss.mp4`

## Contrôles

- **D-PAD LEFT**: Démarrer/Arrêter l'enregistrement vidéo
- **D-PAD RIGHT**: Capturer une photo

## Emplacement des Fichiers

Tous les fichiers (vidéos et photos) sont sauvegardés dans:
- Répertoire par défaut: `~/Documents/`
- Peut être modifié via `videoHandler.setSaveLocation(url)`

## Notes Importantes

1. **Frames disponibles**: Les photos et vidéos ne peuvent être capturées que si le flux vidéo est actif et des frames sont reçues du drone.

2. **Performance**: L'écriture des frames est effectuée de manière asynchrone pour ne pas bloquer l'affichage vidéo.

3. **Synchronisation**: La fonction `displayFrame()` gère à la fois:
   - L'affichage en temps réel
   - L'enregistrement vidéo (si actif)
   - Le stockage de la dernière frame pour la capture photo

4. **Gestion d'erreurs**: Des messages de log détaillés sont affichés pour faciliter le débogage.

## Test

Pour tester les corrections:

1. **Test de l'enregistrement vidéo**:
   - Connectez-vous au drone
   - Appuyez sur D-PAD LEFT pour démarrer l'enregistrement
   - Volez pendant quelques secondes
   - Appuyez sur D-PAD LEFT pour arrêter
   - Vérifiez qu'un fichier .mp4 non-vide apparaît dans Documents
   - Le terminal devrait afficher: "📊 Total frames recorded: X"

2. **Test de la capture photo**:
   - Avec le flux vidéo actif
   - Appuyez sur D-PAD RIGHT
   - Vérifiez qu'un fichier .jpg apparaît dans Documents
   - Le terminal devrait afficher: "📸 Photo saved: ARDrone_Photo_[timestamp].jpg"

## Compatibilité

- macOS 10.15+ (à cause de l'utilisation d'AVFoundation et VideoToolbox)
- Swift 5.0+
- AR.Drone 2.0
