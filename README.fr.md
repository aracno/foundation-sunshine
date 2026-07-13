### ░▒▓ Fonctionnalités principales

- **Chaîne HDR complète** — Encodage double format (PQ + HLG)・Analyse de luminance GPU image par image・Métadonnées dynamiques HDR10+ / HDR Vivid・Transmission complète des métadonnées statiques
- **Moniteur virtuel** — Intégration profonde de [ZakoVDD](https://github.com/qiin2333/zako-vdd)・Emprunt d'image Zako Direct sans copie・5 modes d'écran・Session GUID multi-client
- **Amélioration audio** — Son surround 7.1.4 (12 canaux)・Récupération de perte de paquets Opus DRED・Flux audio continu・Microphone à distance・Correspondance de profondeur de bits du haut-parleur virtuel
- **Optimisation de l'encodage** — NVENC SDK 13.0・AMF QVBR/HQVBR/Instances matérielles multiples・Cache des résultats de l'encodeur (260x)・Sous-échantillonnage adaptatif・Encodeur Vulkan
- **Partage de dossiers** – Mappage de répertoire hôte Windows・Partage par clic droit dans l'Explorateur・Valeurs par défaut sécurisées en lecture seule・Autorisation des appareils appairés
- **Panneau de contrôle** — Tauri 2 + Vue 3 + Vite・Mode sombre・Appairage QR・Surveillance en temps réel・Optimisation du rendu WebUI
- **Entrées améliorées** — Configuration indépendante par client・Adaptation native du pavé tactile de précision・Pilote de souris virtuelle (vmouse)

### ░▒▓ Détails techniques

<details>
<summary><b>Solution technique de la chaîne HDR complète</b></summary>

#### Encodage HDR double format : prise en charge parallèle HDR10 (PQ) + HLG

Les solutions de streaming traditionnelles ne prennent en charge que le mappage de luminance absolue HDR10 (PQ). Lorsque les capacités de l'appareil terminal sont insuffisantes ou que les paramètres de luminance ne correspondent pas, des problèmes de perte de détails dans les ombres et d'écrêtage des hautes lumières peuvent survenir.

Par conséquent, la prise en charge de HLG (Hybrid Log-Gamma, ITU-R BT.2100) a été ajoutée au niveau de l'encodage, en utilisant un mappage de luminance relative :
- **Adaptation de la luminance en fonction de la scène** : HLG est basé sur une courbe de luminance relative. Le terminal d'affichage effectue automatiquement un mappage des tons en fonction de sa propre luminance de crête. La rétention des détails dans les ombres sur les appareils à faible luminance est nettement meilleure qu'avec PQ.
- **Transition en douceur dans les zones de haute luminance** : La fonction de transfert logarithmique-gamma mixte de HLG offre une transition progressive dans les zones de haute luminance, évitant les ruptures de gradation des hautes lumières causées par l'écrêtage brutal de PQ.
- **Rétrocompatibilité SDR naturelle** : Le signal HLG peut être directement décodé par un moniteur SDR en une image standard BT.709, sans nécessiter de traitement de mappage des tons supplémentaire.

**Analyse de luminance image par image et génération adaptative de métadonnées**

Un module d'analyse de luminance en temps réel est intégré côté GPU, exécutant via Compute Shader sur chaque image :
- **Calcul image par image de MaxFALL / MaxCLL** : Statistiques en temps réel de la luminance maximale du contenu par image (MaxCLL) et de la luminance moyenne par image (MaxFALL), injection dynamique dans les métadonnées SEI/OBU HEVC/AV1.
- **Filtrage robuste des valeurs aberrantes** : Utilisation d'une stratégie de troncature par centile pour éliminer les pixels de luminance extrême (par exemple, les reflets spéculaires), empêchant les points lumineux isolés d'augmenter la référence de luminance globale et d'assombrir l'image globale.
- **Lissage exponentiel inter-image** : Application d'un filtrage EMA (moyenne mobile exponentielle) aux valeurs statistiques de luminance des images consécutives pour éliminer le scintillement de luminance causé par les changements brusques de métadonnées lors des transitions de scène.

**Transmission complète des métadonnées HDR**

Les métadonnées statiques HDR10 (Informations d'affichage de mastering + Niveau de lumière du contenu) sont transmises intégralement. Le flux encodé produit par NVENC / AMF / QSV transporte les informations complètes de volume de couleur et de luminance conformes à la norme CTA-861.

**Injection de métadonnées dynamiques HDR10+ / HDR Vivid**

Dans le pipeline d'encodage NVENC, basé sur les résultats de l'analyse de luminance image par image, les métadonnées dynamiques SEI suivantes sont automatiquement générées et injectées :
- **HDR10+ (ST 2094-40)** : Transporte des références de mappage des tons telles que MaxSCL / percentiles de distribution / point de genou par scène, prenant en charge le mappage des tons précis des téléviseurs certifiés HDR10+ Samsung/Panasonic.
- **HDR Vivid (CUVA T/UWA 005.3)** : Norme CUVA (China Ultra-high definition Video Alliance) enregistrée sous ITU-T T.35. Fournit un mappage des tons de luminance absolue en mode PQ et un mappage des tons de luminance relative de référence de scène en mode HLG, couvrant l'écosystème des terminaux nationaux.

</details>

<details>
<summary><b>Intégration du moniteur virtuel</b> (nécessite Windows 10 22H2+)</summary>

Intégration profonde du pilote de moniteur virtuel [ZakoVDD](https://github.com/qiin2333/zako-vdd) :
- Prise en charge des résolutions et fréquences de rafraîchissement personnalisées, profondeur de couleur 10 bits HDR.
- **5 modes de combinaison d'écran** : Écran virtuel uniquement, écran physique uniquement, mode mixte, mode miroir, mode étendu.
- Communication en temps réel via IOCTL, création/destruction automatique du moniteur virtuel au début/fin du streaming.
- Chaque client est lié indépendamment à une session VDD (GUID), prenant en charge la commutation rapide entre plusieurs clients.
- Modification de la configuration en temps réel sans redémarrage.
- **Emprunt d'image Zako Direct sans copie** : Peut emprunter directement la texture d'image partagée VDD, la restituer immédiatement après la conversion, réduisant ainsi la copie GPU dans la chaîne de capture VDD.

</details>

<details>
<summary><b>Amélioration audio</b></summary>

- **Son surround 7.1.4 (12 canaux)** : Mappage complet des canaux pour les formats audio immersifs comme Dolby Atmos.
- **Redondance profonde Opus DRED** : Récupération de perte de paquets basée sur un réseau neuronal, avec une fenêtre de redondance de 100 ms pour compenser en douceur la gigue du réseau.
- **Flux audio continu** : Flux audio ininterrompu, remplissage automatique avec des données silencieuses en l'absence de son, évitant les initialisations répétées du périphérique audio.
- **Correspondance automatique du haut-parleur virtuel** : Détection et correspondance automatiques des formats de profondeur de bits (16 bits/24 bits, etc.) des périphériques audio virtuels.

</details>

<details>
<summary><b>Optimisation de la capture et de l'encodage</b></summary>

**Pipeline de capture**
- **Shader conscient du gamma** : Sélection automatique de la conversion de couleur sRGB / Gamma linéaire en fonction de l'espace colorimétrique DXGI.
- **Sous-échantillonnage de haute qualité** : Interpolation bicubique, prenant en charge trois niveaux : fast / balanced / high_quality.
- **Détection dynamique de la résolution** : Perception en temps réel des changements de résolution et d'orientation du moniteur, l'encodeur s'adapte automatiquement.
- **Analyse de luminance GPU** : Réduction en deux étapes Compute Shader, troncature P95/P99, lissage temporel EMA inter-image.

**NVENC**
- **SDK 13.0** : Contrôle de débit raffiné et Look-ahead.
- **API de métadonnées HDR** : Écriture native des informations d'affichage de mastering / niveau de lumière du contenu via NVENC SDK 12.2+.
- **SEI HDR10+ / HDR Vivid** : Génération automatique image par image des métadonnées dynamiques ST 2094-40 et CUVA T.35.
- **Conformité du flux SPS** : Écriture complète des restrictions du flux binaire SPS H.264/HEVC.

**AMF (AMD)**
- **QVBR / HQVBR / HQCBR** : Contrôle de débit avancé, prenant en charge le réglage de l'interface utilisateur du niveau de qualité.
- **Faible latence contrôlable** : La faible latence AMF, la taille de la file d'attente d'entrée et le mode de latence d'encodage AV1 peuvent être ajustés explicitement dans WebUI, équilibrant une latence extrêmement faible et la stabilité du pilote.
- **Encodage multi-instances matérielles** : Prend en charge les commutateurs liés à AMF Multi-HW Instance / Smart Access Video, permettant au pilote de répartir la charge d'encodage sur les plateformes prises en charge.

**Général**
- **Cache des résultats de l'encodeur** : Persistance des résultats de détection, connexion suivante 26s → <100ms (accélération 260x).
- **Sous-échantillonnage adaptatif** : Prend en charge trois niveaux de mise à l'échelle de la résolution (bilinéaire / bicubique / haute qualité), adapté aux scénarios de streaming hôte 4K → 1080p.
- **Encodeur Vulkan** : Prise en charge expérimentale de l'encodage vidéo Vulkan.
- **Chaîne de certificats sans verrou** : `shared_mutex` remplace mutex, éliminant les frais généraux de la file d'attente TLS.

</details>

<br>

---

### ░▒▓ Clients recommandés

Pour une expérience optimale, associez les clients Moonlight optimisés suivants (activation des propriétés de l'ensemble)

- **PC** — [Moonlight-PC](https://github.com/qiin2333/moonlight-qt) (Windows · macOS · Linux)
- **Android** — [Édition Puissance Renforcée](https://github.com/qiin2333/moonlight-vplus) · [Édition Couronne](https://github.com/WACrown/moonlight-android)
- **iOS** — [VoidLink](https://github.com/The-Fried-Fish/VoidLink-previously-moonlight-zwm)
- **HarmonyOS** — [Moonlight V+](https://appgallery.huawei.com/app/detail?id=com.alkaidlab.sdream)

Plus de ressources : [awesome-sunshine](https://github.com/LizardByte/awesome-sunshine)

<br>

<details>
<summary><b>░▒▓ Configuration système requise</b></summary>

| Composant | Configuration minimale | Recommandé pour 4K |
|------|----------|---------|
| **GPU** | AMD VCE 1.0+ / Intel VAAPI / NVIDIA NVENC | AMD VCE 3.1+ / Intel HD 510+ / GTX 1080+ |
| **CPU** | Ryzen 3 / Core i3 | Ryzen 5 / Core i5 |
| **RAM** | 4 Go | 8 Go |
| **Système** | Windows 10 22H2+ | Windows 10 22H2+ |
| **Réseau** | 5GHz 802.11ac | Ethernet CAT5e |

Compatibilité GPU : [NVENC](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new) · [AMD VCE](https://github.com/obsproject/obs-amd-encoder/wiki/Hardware-Support) · [Intel VAAPI](https://www.intel.com/content/www/us/en/developer/articles/technical/linuxmedia-vaapi.html)

</details>

---

### ░▒▓ Documentation et support

[![Docs](https://img.shields.io/badge/Documentation_d'utilisation-ff69b4?style=flat-square)](https://docs.qq.com/aio/DSGdQc3htbFJjSFdO?p=YTpMj5JNNdB5hEKJhhqlSB) [![LizardByte](https://img.shields.io/badge/Documentation_LizardByte-a78bfa?style=flat-square)](https://docs.lizardbyte.dev/projects/sunshine/latest/) [![QQ群](https://img.shields.io/badge/Groupe_de_discussion_QQ-38bdf8?style=flat-square)](https://qm.qq.com/cgi-bin/qm/qr?k=5qnkzSaLIrIaU4FvumftZH_6Hg7fUuLD&jump_from=webapi)

Vous voulez aider à coder ? → [![Build](https://img.shields.io/badge/Instructions_de_construction-34d399?style=flat-square)](docs/building.md) [![Config](https://img.shields.io/badge/Guide_de_configuration-fbbf24?style=flat-square)](docs/configuration.md) [![WebUI](https://img.shields.io/badge/Développement_WebUI-fb923c?style=flat-square)](docs/WEBUI_DEVELOPMENT.md)

<br>

<div align="center">

« ░▒▓ »

<a href="https://github.com/qiin2333/foundation-sunshine/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=qiin2333/foundation-sunshine&max=100" />
</a>

<br>

[![Rejoindre le groupe QQ](https://pub.idqqimg.com/wpa/images/group.png 'Rejoindre le groupe QQ')](https://qm.qq.com/cgi-bin/qm/qr?k=WC2PSZ3Q6Hk6j8U_DG9S7522GPtItk0m&jump_from=webapi&authKey=zVDLFrS83s/0Xg3hMbkMeAqI7xoHXaM3sxZIF/u9JW7qO/D8xd0npytVBC2lOS+z)

[![Star History Chart](https://api.star-history.com/chart?repos=AlkaidLab/foundation-sunshine&type=date&legend=top-left&sealed_token=8GzivsLWTBiHWFj-MfIXqxD6tKYaPkTgNvC2q8IjHD2nbEypOWmB3bwOGTGtsCNg-ZKW0uy10gX845qiIMElcA4v_qbJh8OUYhiWtI0aSCvempCz97-OcUeWNrYRPz_rZ0hy7mb8Hfj8qnuVAOZ-p04lzSPXNOyVbm4U-acAHIqyQTdm8FXY-jrXzArQ)](https://www.star-history.com/?repos=AlkaidLab%2Ffoundation-sunshine&type=date&legend=top-left)

</div>
```