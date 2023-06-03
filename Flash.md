# SB2040 et le curieux cas de la disparition du firmware

Certaines cartes SB2040 se comportent de manière étrange - les symptômes sont que, après avoir flashé l'image Klipper, elles redémarrent correctement et Klipper est actif (la led gpio24 est allumée et la carte communique via USB/CAN). Cependant, après avoir coupé puis rétabli l'alimentation de la carte, celle-ci se comporte comme si Klipper n'avait pas été flashé.

Certaines personnes ont découvert que Klipper est en réalité présent, et il est possible de le faire démarrer, si vous avez de la chance, en insérant un câble USB-C dans un angle spécifique ou en essayant de l'insérer plusieurs fois, mais cela n'est pas très utile en dehors de prouver que la carte est correctement flashée.

Certaines personnes ont également découvert que CANBOOT sur le RP2040 démarre à chaque fois sans problème. En utilisant le chargeur d'amorçage CANBOOT, il est en réalité possible de flasher la carte de manière à ce qu'elle démarre de manière fiable. Le mécanisme exact de pourquoi une méthode fonctionne et l'autre ne fonctionne pas est inconnu au moment de la rédaction de cette note.

Je documente ici le processus de flashage du CANBOOT et de l'image Klipper.
Notez que je suppose que vous avez au moins une capacité sommaire à travailler via SSH (Putty) et que vous pouvez suivre attentivement les étapes du processus.

# Installation CanBoot

1. Branchez la carte SB2040 en mode bootloader sur l'USB : appuyez sur le bouton de réinitialisation et, tout en le maintenant enfoncé, branchez le câble USB-C.
2. Vérifiez que la carte SB2040 est correctement connectée en mode bootloader en utilisant la commande :
```
lsusb
```
Référez-vous à la capture d'écran suivante :
![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/lsusb.png)

3. Vérifiez que vous voyez la ligne "Raspberry Pi RP2 Boot" en exécutant la commande lsusb. Veuillez vous référer aux remarques si vous voyez le même ID (2e8a:0003), mais que le texte est vide.

Ensuite, nous allons compiler et installer l'image CanBoot :
```
cd ~
git clone https://github.com/Arksine/CanBoot
cd CanBoot
make clean
make menuconfig
```
![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/makemenu1.png)

```
make -j 4
```

```
make flash FLASH_DEVICE=2e8a:0003
```

4. Maintenant, le chargeur d'amorçage CanBoot devrait être installé et actif. Pour le vérifier, observez la LED gpio24 qui doit clignoter lentement. Débranchez le câble USB-C, puis rebranchez-le. La LED devrait recommencer à clignoter.

Quelques remarques/éventuels problèmes (veuillez lire attentivement) :
- Cet exemple suppose que l'interface can0 est configurée avec une vitesse de bauds de 500k. Veuillez modifier la valeur dans le menuconfig pour refléter votre configuration réelle. Assurez-vous d'utiliser la vitesse de bauds choisie de manière cohérente, sinon le réseau CAN ne fonctionnera pas correctement.
- Ce guide ne traite pas de la configuration matérielle, telle que la terminaison de 120 ohms, le câblage des broches, etc.
- Cet exemple suppose que vous souhaitez utiliser votre carte via CAN. Si vous ne prévoyez pas d'utiliser la carte via CAN, vous pouvez modifier la connexion vers USB dans le menuconfig. Vous pouvez toujours reflasher avec une configuration modifiée ultérieurement, donc vous n'êtes pas engagé à quoi que ce soit.
- Si votre carte ne redémarre pas (c'est-à-dire que la LED gpio24 ne se met pas à clignoter), ce guide ne pourra pas vous aider. Vous devriez envisager de retourner la carte.
- Le menuconfig de l'exemple configure le port d'arrêt à deux broches (gpio29) comme un bouton pouvant être utilisé pour forcer l'entrée dans le chargeur d'amorçage CanBoot. Pour ce faire, court-circuitez la broche GND et la broche gpio dans le slot, puis rebranchez le câble USB. Si vous ne prévoyez pas d'utiliser cette fonctionnalité, vous pouvez décocher l'option "Activer l'entrée du chargeur d'amorçage via le bouton (gpio)". Il n'est pas strictement nécessaire d'avoir cette option activée. La dernière version du CanBoot RP2040 initialise la mémoire flash interne de telle manière qu'elle démarre toujours en mode bootloader à moins d'avoir déjà été flashée.
- On m'a dit que certaines SB-2040 démarrent ainsi...

![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/lsusb2.png)

5. Débranchez la carte de l'USB et (avec l'imprimante éteinte), branchez la carte au réseau CAN. Allumez l'imprimante. Vous devriez voir la LED gpio24 recommencer à clignoter.
6. SSH vers l'imprimante.
``` 
cd ~/klipper
../klippy-env/bin/python ../klipper/scripts/canbus_query.py can0
```

![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/uuid.png)

Si aucun UUID n'est trouvé, cela signifie qu'il y a un problème et ce guide ne pourra pas vous aider. Si tout est correct, vous devriez voir (au moins un) UUID avec l'application CanBoot.

Passons à l'étape suivante !

# Configuration et flashage de Klipper :

1. Avec la fusion des modifications nécessaires dans Klipper upstream, nous pouvons simplement utiliser la dernière version de Klipper pour compiler le micrologiciel pour la carte compatible avec CanBoot. Avant d'exécuter make menuconfig, assurez-vous de sauvegarder la configuration précédente (.config). Vous devrez peut-être également mettre à jour le micrologiciel sur les autres cartes si vous n'avez pas mis à jour Klipper depuis un certain temps.
```
cd ~
cd klipper
git pull
make menuconfig
make clean
```

![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/makemenu2.png)

```
make -j 4
```

![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/bin.png)

2. Assurez-vous de voir le message "Création du fichier hex out/klipper.bin". Il est important que le fichier soit klipper.bin, et non klipper.uf2 ! Si vous voyez uf2, revenez à menuconfig et activez l'option "Use CanBoot bootloader".
```
cd ~/klipper
./lib/canboot/flash_can.py -u <UUID> -v -f ./out/klipper.bin
```
(Notez que le chemin dans l'image est légèrement différent et inclut l'UUID de _votre_ système, mais ce n'est pas grave, c'est juste une capture d'écran d'une révision différente du document. La sortie et le comportement seront les mêmes.)

Vérifions si Klipper fonctionne !
Un indicateur est que la LED gpio24 reste constamment allumée (pas de clignotement).
Scannons le bus CAN.
```
cd ~/klipper
../klippy-env/bin/python ./scripts/canbus_query.py can0
```

![image](https://github.com/Eloura74/CanBus_SB2040/blob/main/images/uuid2.png)

3. Éteignez l'imprimante. Allumez l'imprimante. SSH dans le système. Répétez la procédure suivante :
```
cd ~/klipper
../klippy-env/bin/python ./scripts/canbus_query.py can0
```

Maintenant, vous devriez pouvoir utiliser l'UUID (canbus_uuid) dans la configuration de Klipper.



